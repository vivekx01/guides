# How a phone call reaches the agent: the full flow

This guide walks through what happens from the moment someone dials your number to the moment they hear the agent reply. It names each address, port, and protocol, so you can trace a call through the system and find where it stopped if something goes wrong.

Placeholders in `<angle brackets>` stand for your own values:

| Placeholder | Meaning |
|---|---|
| `<sip-domain>` | The SIP subdomain that points at your server, such as `sip.example.com` |
| `<livekit-domain>` | The domain for the LiveKit server, such as `livekit.example.com` |
| `<control-domain>` | The domain for the LiveKit control panel, such as `control.example.com` |
| `<vps-ip>` | Your server's public IP address |
| `<phone-number>` | Your provider's phone number, in E.164 format |

---

## 1. Two kinds of traffic

Every call uses two channels at the same time.

- **Signaling** is the conversation that sets up and ends the call: "I'm calling this number," "accepted," "hanging up." It's short messages, sent once.
- **Media** is the audio. It's a continuous stream of small packets, sent during the whole call.

Signaling and media usually use different protocols and different ports. Most call problems come from one of the two being blocked, so it helps to know which one a symptom points to.

---

## 2. The addresses and ports

| Name | Address | Protocol | Purpose |
|---|---|---|---|
| SIP service | `<sip-domain>`, port 5060 | SIP over TCP and UDP | Receives calls from the phone provider |
| SIP call audio | `<vps-ip>`, UDP 10000–10100 | RTP over UDP | Receives audio from the provider |
| LiveKit server | `wss://<livekit-domain>`, port 443 | WebSocket (signaling) | Rooms, jobs, and the control connection for the SIP service and the agent |
| LiveKit media | `<vps-ip>`, UDP 50000–50100 and TCP 7881 | WebRTC | Audio between the SIP service, the agent, and the room |
| Control panel settings | `https://<control-domain>/api/internal/agents/<agent-name>/settings` | HTTPS | Lets the agent read its settings at the start of each call |
| Speech-to-text | Deepgram's streaming endpoint | WebSocket | Turns caller audio into text |
| LLM | `https://openrouter.ai/api/v1` | HTTPS | Produces the reply text |
| Text-to-speech | Fish Audio's API | HTTPS or WebSocket | Turns reply text into audio |

Check each provider's current endpoint in its documentation. The LiveKit and OpenRouter addresses come from our setup. The Fish address isn't listed here, since the agent reaches it through its plugin.

---

## 3. The flow

### Step 1: The caller dials the number

The caller's phone sends the call to the phone company that owns the number. That's the provider, such as Telnyx.

The provider looks up the SIP connection the number is attached to. The connection says where to send calls: `<sip-domain>`, port 5060.

### Step 2: The provider sends a SIP invite to your server

The provider resolves `<sip-domain>` through DNS. The DNS A record points at `<vps-ip>`.

The provider then sends a **SIP INVITE** to `<vps-ip>` on port 5060. The message says, in effect, "a call is coming for `<phone-number>`." It includes an SDP section that lists the audio format and the address where the provider expects to send audio back.

### Step 3: The SIP service accepts or rejects the call

The SIP service replies with **100 Trying**, to confirm it got the message.

It then looks at the called number and checks the inbound trunks:

- **No matching trunk:** the call is rejected.
- **Matching trunk, allowed addresses set:** the caller's IP must be on the list, or the call is rejected with 403.
- **Matching trunk, credentials set:** the caller must send the matching username and password.
- **Matching trunk, no restrictions:** the call is accepted.

If the call passes these checks, the SIP service moves on to the dispatch rule.

### Step 4: The dispatch rule chooses the room

The SIP service applies the dispatch rule. The rule says what room to use and which agent to bring in.

For an individual rule with prefix `call-`, the room name is `call-` followed by a unique ID. The rule also names the agent, such as `sage`.

The SIP service asks the LiveKit server to create the room. It sends the request over **WebSocket** to `wss://<livekit-domain>`, signed with the API key pair. The server creates the room and returns.

### Step 5: The SIP service joins the room as a participant

The SIP service now needs to put the call into the room. It does this with **WebRTC**, the same protocol browsers use.

1. The SIP service sends a WebRTC offer to the LiveKit server, and the server answers.
2. Both sides exchange possible network paths, called ICE candidates, and pick one that works.
3. The media connection runs over the LiveKit media ports: UDP 50000–50100, with TCP 7881 as a fallback.

Once the room connection is up, the SIP service replies to the provider with **200 OK**. The provider replies with **ACK**. The call is now connected.

### Step 6: Caller audio reaches the room

The provider sends the caller's voice as **RTP** packets to `<vps-ip>` on the SIP media ports, UDP 10000–10100.

The SIP service converts the audio into the format WebRTC uses, and forwards it into the room. From this point, LiveKit treats the caller like any other participant.

### Step 7: LiveKit dispatches the job to the agent

The LiveKit server sees that the new room asks for the `sage` agent. It creates a **job** and sends it to the agent's worker.

The worker already has a WebSocket connection to the LiveKit server, opened when it started. The job arrives over that connection.

### Step 8: The agent joins the room

The worker starts a job process for this room. The process connects to the room the same way the SIP service did: WebSocket for signaling, WebRTC for media, on the same ports.

The agent then links to the caller and starts listening to their audio.

### Step 9: The agent loads its settings

Before the conversation starts, the agent sends an HTTPS request to `https://<control-domain>/api/internal/agents/<agent-name>/settings`, with the internal token.

The control panel returns the greeting, instructions, and model choices. If the request fails, the agent uses its built-in defaults, so the call still works.

### Step 10: The conversation loop

Each turn follows the same path:

1. **Speech to text.** The caller's audio goes from the room to the agent. The agent streams it to the speech-to-text provider over WebSocket. Text comes back as the caller speaks, and the turn-detection model decides when the caller has finished.
2. **Text to reply.** The agent sends the text, plus the conversation so far, to the LLM over HTTPS. The reply comes back in pieces, so speech can start early.
3. **Reply to speech.** The agent sends the reply text to the text-to-speech provider, which returns audio.
4. **Speech back to the caller.** The agent publishes the audio into the room. LiveKit forwards it to the SIP service, which sends it to the provider as RTP. The provider delivers it to the caller's phone.

The voice-activity model, which runs on the agent's machine, detects when someone starts and stops speaking. That tells the agent when to listen and when to talk.

### Step 11: The call ends

When the caller hangs up, their phone sends a **BYE** message. The SIP service tells the room the caller has left.

The agent sees the disconnect and closes its session. The room stays on the server until it times out.

---

## 4. What each part is responsible for

| Part | Responsibility |
|---|---|
| Provider | Owns the number and forwards calls to your server |
| SIP service | Accepts calls, checks the trunk, and turns the call into a room participant |
| LiveKit server | Creates rooms, passes audio between participants, and dispatches jobs |
| Agent | Joins the room and runs the conversation loop |
| Speech-to-text, LLM, text-to-speech | Each does one part of the conversation |
| Control panel | Stores the trunk, dispatch rule, and agent settings |

---

## 5. Where to look when something fails

| What you observe | Where the flow stopped | What to check |
|---|---|---|
| Call doesn't connect at all | Step 2 | DNS for `<sip-domain>`, port 5060 reachable, provider's connection settings |
| Call rings, then the provider reports a rejection | Step 3 | Trunk number matches exactly, allowed addresses, credentials |
| Call connects, no agent | Step 4 or 7 | Dispatch rule exists, its agent name matches the agent, the agent is running |
| Call connects, no audio from the caller | Step 6 | RTP ports 10000–10100 open and published |
| Agent joins, no audio back to the caller | Step 5 or 10 | WebRTC ports 50000–50100 and 7881 open, the agent's connection to the room |
| Agent hears nothing | Step 10, speech to text | The speech-to-text key, network access to the provider |
| Agent replies with defaults | Step 9 | Settings URL and internal token match the control panel |
| Agent is silent | Step 10, LLM or text to speech | The LLM and text-to-speech keys, and their logs |

---

## 6. Why this design

- **Each call gets its own room.** Callers don't hear each other, and each agent job stays separate.
- **The agent doesn't handle phone signaling.** The SIP service does, so the agent only deals with audio in a room. That's why the same agent can serve browsers and phone calls.
- **The settings are read per call.** Changes in the control panel apply to the next call without a redeploy.
- **Each provider does one job.** If one provider has a problem, you can usually swap it without changing the others.

---

## 7. Quick reference

| Item | Value |
|---|---|
| Phone number | `<phone-number>` |
| SIP target | `<sip-domain>`, port 5060 |
| LiveKit server | `wss://<livekit-domain>` |
| Call audio | UDP 10000–10100 |
| WebRTC media | UDP 50000–50100, TCP 7881 |
| Settings endpoint | `https://<control-domain>/api/internal/agents/<agent-name>/settings` |
| LLM endpoint | `https://openrouter.ai/api/v1` |
