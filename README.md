# MeetMind

**An AI assistant that joins your meeting, listens, and answers questions live.**

Paste a Google Meet link. A bot joins the call as a participant, keeps a
speaker-attributed transcript in real time, and answers anyone's questions about
what has been said — in a public channel everyone sees, or a private channel only
you see. When the meeting ends it writes the minutes and drafts the follow-up email.

Built for the ProAdapt Hackathon.

---

## The idea

Existing meeting assistants are locked to one platform. Gemini only works in Google
Meet, Copilot only in Teams. They are features of a product you already pay for, and
using them means asking the meeting host for permission.

MeetMind is built as a **platform, not an extension**. The assistant is a participant
that joins from outside, so nothing has to be installed inside the meeting and no
provider has to approve it.

---

## The key technical decision: captions, not audio

The obvious way to transcribe a meeting is: capture the audio, run Whisper on it, run
a speaker-diarization model to work out who said what.

That is slow, needs a GPU, needs OS-specific audio plumbing, and diarization is
genuinely hard — telling two similar voices apart is an unsolved problem in the
general case.

**MeetMind does not touch audio at all.**

Google Meet already runs Google's speech recognition on the call for its own live
captions. And Meet's caption DOM carries the **speaker's name** next to the text. So
the bot switches captions on and reads them directly out of the page.

What that buys us:

| | Audio + Whisper + diarization | Caption scraping |
|---|---|---|
| Speaker attribution | a separate ML model, often wrong | **free and exact** |
| Hardware | GPU strongly preferred | **any laptop** |
| Setup | model downloads, audio drivers | **nothing** |
| Latency | transcribe after the fact | **live** |
| Accuracy | whatever model you can run locally | **Google's production ASR** |

The trade-off is honest: this only works where the platform provides captions. That
is why Whisper stays on the roadmap — as the fallback for platforms that do not, not
as the primary path.

---

## Architecture

```
  Google Meet
       │  (Playwright drives a real Chromium in as a guest)
       ▼
  ┌─────────────┐   captions {speaker, text}
  │   bot.ts    │──────────────────────────────┐
  └─────────────┘                              ▼
                                         ┌───────────┐
   browser UI  ◄────── WebSocket ────────►│ server.ts │
  (transcript,                            └───────────┘
   public chat,                                 │
   private chat)                                ▼
                                          Groq · GPT-OSS 120B
```

One Node process. State lives **in memory** — there is no database, and the
transcript dies with the process. That is deliberate: nothing is retained, which is
also the honest answer to "is this private?".

### How each piece works

**`src/bot.ts` — getting into the meeting**

Launches a real Chromium through Playwright, **headful**, because Meet detects and
blocks headless browsers. It joins muted and with the camera off, using fake media
devices so no real microphone is needed.

It handles the states Meet actually produces: a guest-name prompt, "Ask to join",
"Join now", and — when the bot's account is already registered in the call from a
previous run — "Switch here" / "Join here too".

It also **verifies it is still on the meeting URL** after loading and again after
clicking join. Google silently redirects signed-out browsers to its marketing page;
without that check the bot reports "waiting to be admitted" while sitting on a
completely different website.

**`src/captions.ts` — reading the captions**

A `MutationObserver` injected into the Meet page. Two details make this work:

1. *It is shipped as a source string, not a function.* `tsx` compiles with esbuild's
   `keepNames`, which wraps functions in a `__name` helper. Playwright sends a
   function to the browser by stringifying it, and `__name` does not exist there —
   injection fails with `ReferenceError: __name is not defined`.

2. *Meet revises captions after showing them.* `"My name is."` becomes
   `"Hello, Datta. My name is."` a second later. The text is **not append-only**, so a
   delta cannot be computed. Each caption block therefore gets one stable key and the
   consumer **overwrites that line in place** — otherwise the transcript repeats itself.

The container is found by ARIA role and label rather than by class name, because
Meet's class names are obfuscated and change without notice. Buttons and Material
icon ligatures inside the captions region (`arrow_downward`, "Jump to bottom") are
filtered out so UI chrome never lands in the transcript.

**`src/server.ts` — fan-out and routing**

Express plus a WebSocket server. Public chat is broadcast to every connected client;
a private reply is sent **only to the socket that asked**, so the separation is real
rather than cosmetic.

It enforces **one assistant per server**: if a bot is already live, a second join
request returns `alreadyRunning` instead of putting a second bot in the call.

**`src/llm.ts` — the only place that talks to a model**

Swapping Groq for Gemini, OpenAI or a local Ollama means editing this one file.

Note `reasoning_effort: 'low'`. GPT-OSS is a reasoning model — without it, the model
spends its whole token budget thinking and returns empty content. With it, answers
come back in **under a second**, which matters when the latency is on a projector.

---

## Features

- **Live transcript** with speaker names and timestamps. Lines appear greyed while
  someone is still speaking, then settle once they pause.
- **Public chat** — ask the assistant something and everyone in the meeting sees the
  answer.
- **Private chat** — ask the same assistant privately. Nobody else sees it, and the
  bot keeps transcribing the meeting throughout.
- **Catch-up questions** — *"What did I miss?"*, *"What did we decide?"*,
  *"What are my action items?"* Answers are grounded in the transcript and attributed
  to whoever said it.
- **Minutes** — summary, decisions, action items with owner and deadline, and open
  questions.
- **Follow-up email** — a ready-to-send draft with subject, decisions,
  `Name - task - deadline` per person, and open questions. Copy it, or open it
  pre-filled in your mail client.

The assistant is told to answer **only** from the transcript and to say plainly when
something has not come up. Empty decisions and action items are correct output for a
meeting where nobody decided anything — it will not pad the minutes.

---

## Setup

Needs **Node.js 18+**.

```bash
npm install            # also downloads the browser the bot drives
cp .env.example .env   # Windows: copy .env.example .env
```

Put a **free Groq API key** in `.env` (console.groq.com → API Keys, no card needed):

```
GROQ_API_KEY=gsk_your_key_here
```

Sign the bot's browser profile into a Google account — **once per machine**:

```bash
npm run login
```

A window opens; sign in and it closes itself. **Do not skip this.** A signed-out bot
gets redirected off the meeting by Google and never arrives.

Then:

```bash
npm start              # http://localhost:3000
```

---

## Running it

### Solo
Everything on one machine: one server, one bot, one assistant.

### Together (several people, one assistant)
**One person hosts.** They run `npm start` and click *Send the bot in*. The terminal
prints an address like `http://192.168.1.20:3000`, which also appears in a green bar
in the app with a Copy button.

**Everyone else opens that address in a browser.** They do not run their own copy.

Five people each starting their own bot would put five bots in the call, each with
its own transcript and its own memory — five assistants, not one. The server refuses
to start a second bot, so this is safe even if someone clicks out of habit.

Everyone then shares one transcript and one public chat, while each person keeps
their **own** private channel.

*Requires everyone on the same network.* Many venue and campus networks isolate
clients from each other, in which case use solo mode or a tunnel.

---

## The demo

1. Start a Meet in your normal browser. Stay in the call — you are the host.
2. Paste the link into MeetMind, click **Send the bot in**.
3. **Admit** `AI ASSISTANT` when it asks to join.
   *The host has to let it in — it cannot enter a meeting uninvited.*
4. Talk. Names and text appear live.
5. Ask something in **Public**, then the same in **Private 🔒**.
6. **Summary** → decisions, owners, deadlines.
7. **Follow-up ✉** → the email, ready to send.

**For the summary to have content, say decision-shaped things out loud** — *"Decision:
we ship Friday. Nakul owns the backend, done by 3 PM."* The model will not invent
tasks nobody assigned.

---

## Files

| File | Role |
|---|---|
| `src/bot.ts` | Joins the Meet, turns captions on |
| `src/captions.ts` | Runs inside the Meet page, scrapes captions |
| `src/server.ts` | REST + WebSocket, public/private routing, single-bot guard |
| `src/llm.ts` | The only code that talks to a model |
| `src/prompts.ts` | Q&A, minutes and follow-up prompts |
| `src/store.ts` | In-memory transcript and chat |
| `scripts/login.mjs` | One-time Google sign-in for the bot profile |
| `public/` | The whole UI — no framework, no build step |

No React and no bundler: a build step is a thing that can break, and this has to work
on four laptops under time pressure.

---

## Troubleshooting

**Bot never appears in the meeting.**
It is signed out. Stop the server and run `npm run login`. Google redirects
signed-out browsers to its marketing page. The app now detects this and says so.

**Transcript stays empty.**
Captions are off. In the *bot's* Chromium window click **CC**, or press `c`.

**"Could not find the join button."**
The bot's window is visible on purpose — click it yourself. Everything downstream
still works.

**Assistant replies with a warning instead of an answer.**
`GROQ_API_KEY` is missing or wrong in `.env`. Check `/api/health` — it reports
`hasKey`.

**Teammates cannot open the shared address.**
Different network, or the firewall is blocking port 3000. Check you are all on the
same wifi.

---

## Limitations

- **Google Meet only.** The architecture is provider-agnostic — only `bot.ts` is
  Meet-specific — but the joiner is written for Meet.
- **Depends on captions**, so it depends on Meet's DOM. Class names are avoided in
  favour of ARIA roles, but Google can still change it.
- **Nothing is persisted.** Restarting the server loses the meeting.
- **One meeting at a time** per server.
- Transcription is only as good as Google's ASR — accents and crosstalk still bite.

---

## Roadmap

- Zoom and Teams joiners (same scraper, different join flow)
- Whisper fallback for platforms without captions
- Sending the follow-up from the assistant's own account — attendee addresses would
  come from the **Google Calendar API**, since Meet exposes display names, not emails
- Spoken replies in the meeting
- Cross-meeting memory: *"what did we decide about pricing last month?"*
- Jira / Slack / Notion push for action items

---

## On privacy

Worth being straight about, because it is the thing people distrust in this category:

- The bot **cannot enter without the host admitting it**, and it appears in the
  participant list under its own name.
- It joins **muted, with the camera off**.
- Nothing is stored. No database, no recordings — the transcript is in memory and
  dies with the process.
- The follow-up email is **drafted, never sent automatically**. A human stays between
  the meeting and anyone's inbox.

---

## Stack

Node.js · TypeScript · Playwright · Express · WebSocket · Groq (GPT-OSS 120B) ·
vanilla JS front end
