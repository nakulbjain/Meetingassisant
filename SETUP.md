# MeetMind — setup & demo runbook

An AI attendee that joins a real Google Meet, keeps a speaker-attributed live
transcript, answers questions in public **and** private channels, and writes the minutes.

Every laptop runs the whole stack. Nobody depends on anyone else's machine.

---

## 1. Setup (about 3 minutes)

You need **Node.js 18+** (`node -v` to check; get it from nodejs.org if missing).

```bash
npm install          # also downloads the browser the bot drives
```

Then get a **free Groq API key** — each person needs their **own**, or you will hit
rate limits mid-interview:

1. Go to **console.groq.com** → sign in → *API Keys* → *Create API Key*
2. Copy it (starts with `gsk_`)
3. Open the `.env` file in this folder and put it on the first line:

```
GROQ_API_KEY=gsk_your_actual_key_here
```

No credit card, takes two minutes.

## 2. Run

```bash
npm start
```

Open **http://localhost:3000**.

---

## 2b. Two ways to run it

### Solo (your own interview)
You run everything on your own laptop. One server, one bot, one assistant.
This is the mode each of you uses when interviewed alone. Nothing extra to do.

### Shared (demoing together, 5 people)
**One laptop hosts the assistant. Everyone else just opens its address.**

There must only ever be ONE bot in a meeting. Five people each clicking
"Send the bot in" would put five bots in the call, each with its own transcript
and its own memory - five assistants, not one.

So:

1. **One person** (the host) runs `npm start` and clicks **Send the bot in**.
2. The terminal prints a line like `http://192.168.26.217:3000` - that address
   also appears in a green bar inside the app, with a **Copy** button.
3. **Everyone else opens that address in their browser.** They do not run
   `npm start`, and they do not click join - the button is disabled for them
   because the assistant is already live.

Everyone then shares one transcript and one public chat, and each person still
gets their **own private channel**, because private replies are routed to the
individual browser that asked - not stored per-machine.

The server refuses to start a second bot, so this is safe even if someone
clicks the button out of habit.

**Requirement:** everyone must be on the same wifi. If the venue's guest network
isolates devices from each other, fall back to solo mode - or ask and we can add
a Cloudflare tunnel for a public link.

---

## 3. The demo, step by step

1. **Start a Google Meet** in your normal browser. Keep that tab open — you are the host.
2. Copy the meeting link, paste it into MeetMind, hit **Send the bot in**.
3. A second Chromium window opens and asks to join. **In your Meet tab, click Admit.**
   - Say this out loud to the judges: *"The host admits it — consent is explicit,
     it never joins a meeting uninvited."*
4. **Speak.** Your words appear in the transcript **with your name**, greyed while
   you are still talking, solid once you pause.
5. Ask it something in the **Public** tab — *"What did we decide?"* Everyone sees it.
6. Switch to **Private 🔒** and ask *"What are my action items?"* — only you see the
   answer, and the bot keeps transcribing the whole time.
7. Hit **Summary** → decisions, action items with owners and due dates, open questions.
8. Hit **Follow-up ✉** → a ready-to-send email drafted from the meeting: subject,
   decisions, "Name - task - deadline" per person, and open questions. Fill in the
   **To** field, click **Open in email client**, and your mail app opens pre-filled.
   - Worth saying out loud: *"A human stays between the meeting and anyone's inbox.
     Nothing is sent automatically."*

### If the transcript stays empty
Captions are not on. In the **bot's** Chromium window, click the **CC** button
(bottom bar) or press **c**. The transcript starts flowing immediately.

### If the bot cannot find the join button
Its browser window is visible on purpose — click **Ask to join** yourself. Everything
downstream still works.

---

## 4. How it works (learn this — the interview asks *you*, alone)

> MeetMind is a Node/TypeScript server that drives a real Chromium browser into a
> Google Meet as a guest using Playwright. The insight is that **we never touch audio.**
> Meet already runs Google's speech recognition for its own live captions, and its
> caption DOM carries the *speaker's name* next to the text. So the bot switches
> captions on and scrapes them with a MutationObserver. That gives us a
> speaker-attributed transcript with no Whisper, no speaker-diarization model, no GPU,
> and no audio plumbing — it runs on any laptop.
>
> Captions grow in place while someone talks, so each caption block is tracked
> individually: we stream partial updates for the live feel and commit a final line
> once a block stops changing for ~1.8s.
>
> Lines flow into an in-memory transcript and out over WebSocket to every connected
> browser. Questions go to GPT-OSS 120B on Groq with the transcript as context.
> Public answers broadcast to everyone; private answers are sent only to the socket
> that asked — the server never puts them on the wire to anyone else.

**Architecture in one line:**
`Meet → Playwright → caption observer → server (in-memory) → WebSocket → browser UI`
with `Groq (GPT-OSS 120B)` hanging off the server for question answering.

### Files worth knowing
| File | What it does |
|---|---|
| `src/bot.ts` | Joins the Meet, turns on captions |
| `src/captions.ts` | Runs *inside* the Meet page, scrapes captions |
| `src/server.ts` | REST + WebSocket, routes public vs private |
| `src/llm.ts` | The only place that talks to a model — swap providers here |
| `src/store.ts` | In-memory transcript and chat |
| `public/app.js` | Transcript + chat client |
| `public/followup.js` | Follow-up email drafting |

### Honest answers to likely questions
- **"Why not Whisper?"** It is the roadmap path for platforms that do not offer
  captions. For Meet it would be strictly worse: Google's ASR is already running on
  the call and already knows who is speaking, so re-transcribing the audio ourselves
  would cost latency and accuracy to arrive at the same answer.
- **"How do you know who said what without a diarization model?"** We do not need
  one - Meet's caption DOM carries the speaker's name next to the text.
- **"Does this work on Zoom and Teams?"** The architecture is provider-agnostic —
  only `bot.ts` is Meet-specific. Teams and Zoom both expose live captions with
  speaker names; it is the same scraper against a different join flow.
- **"Is this secure/private?"** Nothing is stored — the transcript lives in memory and
  dies with the process. No database. The bot cannot enter without the host admitting it.
- **"Can it send the email itself?"** Deliberately not yet. Sending from the bot's
  own Gmail needs either OAuth or an App Password, and auto-emailing everyone in a
  call is exactly the behaviour people distrust. Attendee addresses would come from
  the Google Calendar API (the event's attendee list) - Meet itself only exposes
  display names, not emails.
- **"What's next?"** Zoom/Teams joiners, Whisper for platforms without captions,
  sending the follow-up from the bot account, voice replies, cross-meeting memory,
  Jira/Slack push.

---

## 5. Testing without a meeting

Inject fake transcript lines to exercise chat and summary:

```bash
curl -X POST http://localhost:3000/api/dev/utterance \
  -H "content-type: application/json" \
  -d "{\"speaker\":\"Ravi\",\"text\":\"Lets ship on Friday.\"}"
```

---

## 6. Getting the code to your teammates

**Never share these three things:**

| Do not send | Why |
|---|---|
| `.env` | Your Groq API key. Everyone needs their own or you hit rate limits. |
| `bot-profile/` | **A logged-in Google session.** Sending it hands over account access. |
| `node_modules/` | 60 MB of files `npm install` recreates in two minutes. |

`.gitignore` already excludes all three, and `MeetMind-share.zip` is built without them.

### Option A - the zip (no accounts, works offline)
Send `MeetMind-share.zip` (39 KB) over WhatsApp, email or a USB stick.

### Option B - a private GitHub repo
```bash
git init
git add .
git commit -m "MeetMind - live AI meeting assistant"
gh repo create meetmind --private --source=. --push
```
Then teammates run `gh repo clone <your-username>/meetmind`.

### What each teammate does after unzipping or cloning
```bash
npm install            # also downloads the browser the bot drives (~2 min)
copy .env.example .env # then paste THEIR OWN Groq key into it
npm run login          # sign in with a Google account - once per laptop
npm start
```

**The step people forget is `npm run login`.** Without it Google bounces the bot
to a marketing page and it never reaches the meeting. Do this on every laptop
*before* the interviews, not during one.
