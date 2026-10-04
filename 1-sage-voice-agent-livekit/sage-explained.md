# Sage, explained simply

This document explains how the whole voice agent works, from a phone call or browser to Sage's reply. It uses plain language first, then shows the code that does each job.

Items marked **(check)** are parts we haven't tested against the installed version yet.

---

## 1. The one-minute version

- **LiveKit** is a room service. It creates rooms and carries audio between people in them.
- **Sage** is a program that joins a room, listens, thinks, and talks back.
- **Providers** do the individual jobs: Deepgram hears, OpenRouter thinks, Fish speaks.

LiveKit moves the audio. The providers do the thinking. Sage connects them.

---

## 2. The pieces, and what each one does

| Piece | Plain-English job | Where it runs |
|---|---|---|
| **LiveKit server** | Creates rooms and passes audio between people in them | Your VPS |
| **SIP service** | Turns a phone call into audio in a room | Your VPS (later) |
| **Sage** | Joins the room, runs the conversation | Your PC for now |
| **Deepgram** | Turns speech into text | Cloud |
| **OpenRouter** | Writes the reply text | Cloud |
| **Fish** | Turns reply text into speech | Cloud |
| **Silero** | Detects when someone is speaking | Your PC |

---

## 3. Keys: who needs what

You have one **LiveKit key pair** (an API key and a secret) and several **provider keys**. They're separate.

```
LIVEKIT_API_KEY=...      # identifies your LiveKit server
LIVEKIT_API_SECRET=...   # signs tokens and proves Sage is allowed to join
DEEPGRAM_API_KEY=...     # for speech to text
OPENROUTER_API_KEY=...   # for the reply
FISH_API_KEY=...         # for speech
```

The LiveKit key pair is used by the server, by Sage, by the token script, and (later) by the SIP service and Egress. Provider keys are only used by Sage.

Keep all of these in `.env`. Never share them or commit them.

---

## 4. Rooms and tokens

A **room** is a place where people and agents exchange audio.

A **token** is a signed pass that says:
- who you are (your *identity*),
- which room you can enter,
- whether you can talk and listen,
- whether an agent should be sent to the room with you.

Here's how we make one. This is `make_token.py`:

```python
import datetime
import os
from livekit import api

token = (
    api.AccessToken(os.environ["LIVEKIT_API_KEY"], os.environ["LIVEKIT_API_SECRET"])
    .with_identity("alice")                       # your unique name in the room
    .with_grants(api.VideoGrants(
        room_join=True,
        room="sage-test",                         # which room you may enter
        can_publish=True,                         # you may send audio
        can_subscribe=True,                       # you may hear others
    ))
    .with_room_config(api.RoomConfiguration(
        agents=[api.RoomAgentDispatch(agent_name="sage")],  # send Sage to this room
    ))
    .with_ttl(datetime.timedelta(hours=2))        # the pass expires
    .to_jwt()
)
```

Run it with:

```powershell
uv run python make_token.py sage-test alice
```

**Rules to remember:**
- Every person needs a **different identity**. If two people use the same one, the first gets kicked out.
- A token only works for the room it names, and only until it expires.
- Anyone with a valid token can join, so treat tokens like passwords.

---

## 5. How Sage connects to the server

Sage doesn't wait for calls. It connects to the server and says, "I'm available for the `sage` agent." It does this by opening a connection *out* from your PC, so you don't need to open any ports on your PC.

Here's the part of `sage.py` that does this:

```python
from livekit import agents
from livekit.agents import AgentServer

server = AgentServer()

@server.rtc_session(agent_name="sage")   # the name the server uses to find Sage
async def sage_session(ctx: agents.JobContext):
    ...   # this runs each time Sage is sent to a room

if __name__ == "__main__":
    agents.cli.run_app(server)           # connects to the server and waits
```

Start it with:

```powershell
uv run python sage.py dev
```

Once it's running, the log shows:

```
registered worker {"agent_name": "sage", "url": "wss://livekit.<your-domain>"}
```

Sage is now available. It doesn't join any room yet.

---

## 6. How Sage gets into a room

Sage joins only when someone with a dispatch token enters a room:

1. You join `sage-test` with a token that includes `agent_name="sage"`.
2. The server finds the registered Sage worker.
3. The server sends Sage a **job** for that room.
4. Sage starts a job process and joins the room.
5. Sage **links to one person** in the room (the first one, by default) and starts listening.

**Two rules to remember:**
- **Dispatch happens when the room is created.** If the room already exists and Sage left, a new join won't bring Sage back. Use a new room name for a fresh session.
- **Sage listens to one person.** Others can hear the conversation, but Sage replies only to the linked person.

---

## 7. Sage's brain: the voice pipeline

Inside `sage_session`, we set up the parts Sage uses for each turn:

```python
from livekit.agents import AgentSession, Agent, TurnHandlingOptions
from livekit.plugins import deepgram, fishaudio, openai, silero

class Sage(Agent):
    def __init__(self):
        super().__init__(instructions="You are Sage. Keep replies short. No lists or emojis.")

session = AgentSession(
    stt=deepgram.STTv2(model="flux-general-en", eager_eot_threshold=0.4),   # hear
    llm=openai.LLM.with_openrouter(model="openai/gpt-4o-mini"),              # think
    tts=fishaudio.TTS(model="s2.1-pro-free", voice_id="933563129e564b19a115bedd57b7406a"),  # speak
    vad=silero.VAD.load(),                                                  # detect speech
    turn_handling=TurnHandlingOptions(turn_detection="stt"),                # know when you're done
)

await session.start(room=ctx.room, agent=Sage())
await session.generate_reply(instructions="Greet the user briefly.")
```

Here's what each line does:

| Line | Job | Simple version |
|---|---|---|
| `stt=` | Speech to text | Writes down what you say |
| `llm=` | The brain | Decides what to say back |
| `tts=` | Text to speech | Says the reply out loud |
| `vad=` | Voice detection | Notices when someone starts or stops talking |
| `turn_handling=` | Turn-taking | Decides when your turn is finished |

---

## 8. One turn, step by step

When you say "What's the weather?":

1. **Silero** notices speech has started.
2. **Deepgram** turns your words into text as you speak.
3. **Flux** decides you've finished your turn.
4. The text goes to **OpenRouter**, which writes a reply.
5. The reply goes to **Fish**, which turns it into speech.
6. Sage publishes that speech into the room.
7. You hear it.

This repeats for every turn. If you interrupt Sage, it stops and listens again.

---

## 9. Swapping the brain (LLM)

The `llm=` line is the only part that decides how Sage thinks. You can replace it with any of these, without changing anything else:

```python
# OpenRouter (what we use now)
llm = openai.LLM.with_openrouter(model="openai/gpt-4o-mini")

# Anthropic's plugin
from livekit.plugins import anthropic
llm = anthropic.LLM(model="claude-sonnet-4-6")

# A LangGraph graph, through LiveKit's LangChain plugin
# (installs as livekit-plugins-langchain)
llm = LLMAdapter(graph)   # adapter from the plugin, graph built with LangGraph
```

The rest of the pipeline doesn't change.

---

## 10. Using your own custom brain

If you have your own harness, it can be the `llm=` value as long as it follows LiveKit's rules. LiveKit needs two things from it:

1. **A class that inherits from LiveKit's LLM base class**, with `model` and `provider` set.
2. **A `chat()` method** that takes the conversation and returns a stream of text pieces.

Here's the outline. This is **(check)**: confirm the exact names in the installed package before you write it, using the OpenAI plugin source in `.venv` as a template.

```python
from livekit.agents import llm

class MyHarnessLLM(llm.LLM):
    def __init__(self, provider_name: str):
        super().__init__()
        self._provider_name = provider_name

    @property
    def model(self) -> str:
        return "my-harness"

    @property
    def provider(self) -> str:
        return self._provider_name

    def chat(self, *, chat_ctx, tools=None, conn_options=None, **kwargs):
        # Return an LLMStream subclass that calls your harness
        # and sends text chunks back as they arrive.
        ...
```

Inside your harness, you can use any providers you like. LiveKit only sees the text that comes out.

Passing it in looks the same as any other brain:

```python
session = AgentSession(
    stt=...,
    llm=MyHarnessLLM("anthropic"),
    tts=...,
    vad=...,
)
```

---

## 11. Quick tips for speed

- **Keep replies short.** Long replies take longer to speak, and the caller waits longer for the first word.
- **Stream text early.** Start speaking as soon as the first words arrive, rather than waiting for the full reply.
- **Keep tool calls quick.** Each slow step before the first word adds to the wait.
- **Test with one person first.** Add more people only after the single-person test feels right.

---

## 12. The main things to remember

1. **LiveKit** moves audio. It doesn't do the thinking.
2. **Tokens** let people into rooms. Each person needs a different identity.
3. **Sage** registers once, then joins a room only when dispatched.
4. **Dispatch happens at room creation.** Use a new room name for a fresh session.
5. **Sage listens to one person** at a time.
6. **Providers** do the individual jobs, and any one of them can be swapped.
7. **Your own harness** works if it follows LiveKit's rules for the LLM.

---

## 13. Related documents

- `how-livekit-agents-work.md`: deeper explanation of registration, tokens, and dispatch
- `livekit-self-host-coolify.md`: setting up the LiveKit server on Coolify
- `livekit-egress.md`: recording calls and rooms
