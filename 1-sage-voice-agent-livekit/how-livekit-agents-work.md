# How the Sage voice agent works with LiveKit

This document explains how Sage registers with the LiveKit server, how tokens let people and agents join rooms, and how a call flows through the system. It's written for someone new to the setup, with the technical details included.

Placeholders in `<angle brackets>` stand for your own values. Never write real keys or tokens into documents or chats.

---

## 1. The big picture

Three kinds of things take part:

| Part | Where it runs | What it does |
|---|---|---|
| **LiveKit server** | Your VPS (Coolify) | Creates rooms, routes audio between participants, checks tokens, assigns jobs to agents |
| **Sage (agent worker)** | Your Windows PC (for now) | Joins rooms when dispatched, listens, thinks, and speaks |
| **Participants** | Browsers, or phone calls via SIP later | People who talk to Sage |

The AI services (Deepgram, OpenRouter, Fish) run in the cloud. Each one does a single job, and LiveKit only moves audio between the parts.

```
Person (browser)  ──WebRTC──▶  LiveKit server (VPS)  ◀──WebSocket──  Sage worker (PC)
                                     │                                  │
                                     │ audio routed to the room         │ calls
                                     ▼                                  ▼
                                 same room                   Deepgram (speech→text)
                                                             OpenRouter (text→reply)
                                                             Fish (reply→speech)
```

---

## 2. Keys: the foundation

You generated one **key pair**: an API key (a public identifier) and an API secret (a private signing key). The same pair is used in three places:

1. **LiveKit server config** (`LIVEKIT_KEYS` in Coolify). The server knows the pair is valid.
2. **Sage's `.env`** (`LIVEKIT_API_KEY`, `LIVEKIT_API_SECRET`). Sage uses it to register.
3. **The token script** (`make_token.py`). It uses the pair to sign tokens for people.

The key pair is only for LiveKit. It has nothing to do with Deepgram, OpenRouter, or Fish, which each have their own keys.

---

## 3. Agent registration

Sage registers with the server by opening a connection out from your PC. Nothing on your PC needs a port open.

### Steps

1. **Start the worker:** `uv run python sage.py dev` reads `.env` and loads the plugins.
2. **Connect:** the worker opens a WebSocket to `LIVEKIT_URL` (`wss://livekit.<your-domain>`).
3. **Authenticate:** the worker signs in with the key pair.
4. **Announce itself:** the worker says it handles the agent named `sage`. That name comes from this line in `sage.py`:
   ```python
   @server.rtc_session(agent_name="sage")
   ```
5. **Registered:** the server records the worker as available. The log shows this:
   ```
   registered worker {"agent_name": "sage", "url": "wss://livekit.<your-domain>"}
   ```

Registration means "Sage is available." It doesn't join any room.

---

## 4. Tokens: how someone joins a room

A **token** is a signed statement that says who the person is, which room they may enter, and what they can do. The server checks the signature with the secret and lets them in only if it matches.

Our token script is `make_token.py`. It builds a token with these parts:

| Part | Code in `make_token.py` | Meaning |
|---|---|---|
| Identity | `.with_identity("alice")` | A unique name for this person in the room |
| Room access | `api.VideoGrants(room_join=True, room="sage-test")` | May join `sage-test` |
| Publish and listen | `can_publish=True, can_subscribe=True` | May send and hear audio |
| Agent dispatch | `api.RoomConfiguration(agents=[api.RoomAgentDispatch(agent_name="sage")])` | When this person joins, ask the `sage` agent to join too |
| Lifetime | `.with_ttl(datetime.timedelta(hours=2))` | Expires after 2 hours |

Generate a token:

```powershell
uv run python make_token.py <room-name> <identity>
```

Tokens are saved to a file so they don't appear in chats.

### Why identity matters

- Each person needs a **different identity**. If two people join with the same identity, the server removes the first one. That's why you were kicked out when your friend joined with the same token.
- Identities are checked by the server, so they can't be changed after the token is issued.

### Who can join

Anyone holding a valid, unexpired token for that room can join. Treat tokens like passwords.

---

## 5. Dispatch: how Sage gets into a room

Dispatch is how the server decides which agent should join a room.

1. Someone joins a room with a token that includes the `sage` dispatch.
2. The server sees the dispatch request and finds a registered worker for `sage`.
3. The server sends the worker a **job** for that room.
4. The worker starts a **job process**, which joins the room as a participant.
5. Sage's session attaches to the person in the room. The log shows:
   ```
   RoomIO linked to participant {"participant_identity": "tester", "room_name": "sage-test"}
   ```

### Important behavior

- **Dispatch happens when a room is created.** If the room already exists and Sage has left, a new join won't bring Sage back. That's why we use a new room name for a fresh session.
- **Sage listens to one participant at a time.** It links to a single person. Others in the room can hear the conversation but Sage won't respond to them.
- **Sage leaves when its person leaves.** The log showed `closing agent session due to participant disconnect` when `tester` left.

---

## 6. A full call, step by step

Using the browser test:

1. You run the token script for a room (for example `sage-test`) with your own identity. The token includes a `sage` dispatch.
2. You open Meet, set the server to `wss://livekit.<your-domain>`, and paste the token.
3. You join. The server creates the room and dispatches the `sage` job.
4. Sage's worker, already registered, starts a job process and joins the room.
5. Sage links to you and starts listening.
6. Your speech goes to Silero (voice detection) and then to Deepgram (speech to text).
7. The text goes to OpenRouter, which returns a reply.
8. The reply goes to Fish, which returns speech.
9. Sage publishes that speech into the room. The server sends it to you.

Steps 6 to 9 repeat for each turn.

---

## 7. What's not in place yet

- **Phone calls.** Twilio and the SIP service aren't connected. Phone calls would arrive through SIP and join a room the same way a browser does.
- **Sage on the VPS.** Sage runs on your PC, so it stops when the PC is off.
- **Multiple listeners.** Sage handles one person per room.
- **Redis.** The current server runs in single-node mode, so Redis isn't set up.

---

## 8. Quick reference

| Task | Command or location |
|---|---|
| Start Sage | `uv run python sage.py dev` |
| Start Sage for production-style running | `uv run python sage.py start` |
| Test Sage on your microphone | `uv run python sage.py console` |
| Make a token | `uv run python make_token.py <room> <identity>` |
| Server URL | `wss://livekit.<your-domain>` |
| Keys | `.env` (never share, never commit) |
| Logs | Sage's terminal output |

---

## 9. Glossary

- **Room:** a session where participants and agents exchange audio.
- **Participant:** anyone in a room, human or agent.
- **Identity:** the unique name a participant uses in a room.
- **Token:** a signed, time-limited permit to join a room.
- **Dispatch:** a request to bring an agent into a room.
- **Worker:** the Sage process that registers and accepts jobs.
- **Job:** one agent session for one room.
- **WebRTC:** the real-time audio protocol used between browsers and the server.
- **SIP:** the protocol phone networks use. We'll use it later for phone calls.
