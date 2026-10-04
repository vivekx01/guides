# Sage voice agent with self-hosted LiveKit

Guides for running a phone-capable voice agent on your own server: LiveKit for rooms, a SIP service for phone calls, and Sage, a Python agent that talks through speech-to-text, an LLM, and text-to-speech.

Every guide uses placeholders in `<angle brackets>` for your own IP, domain, keys, and passwords. Never commit real values.

## Guides

| Guide | What it covers |
|---|---|
| [sage-explained.md](sage-explained.md) | Start here. The whole system in plain language, with code examples. |
| [how-livekit-agents-work.md](how-livekit-agents-work.md) | How agents register, how tokens let people join rooms, and how dispatch brings an agent into a room. |
| [livekit-self-host-coolify.md](livekit-self-host-coolify.md) | Running the LiveKit server on a VPS with Coolify: swap, keys, ports, and verification. |
| [livekit-sip-service.md](livekit-sip-service.md) | Running the SIP service, the DNS record, the trunk and dispatch rule, and connecting a phone provider. |
| [trunks-and-dispatch-rules.md](trunks-and-dispatch-rules.md) | Complete guide to inbound trunks and dispatch rules, with setup options and provider configuration (Twilio, Telnyx). |
| [livekit-softphone-test.md](livekit-softphone-test.md) | Testing the full call path with a free softphone, without a phone carrier. |
| [livekit-egress.md](livekit-egress.md) | Recording rooms and tracks with LiveKit Egress: what it does, what it needs, and how we could use it. |

## Suggested order

1. `livekit-self-host-coolify.md`: get the LiveKit server running.
2. `livekit-sip-service.md`: add phone calls, starting with the DNS record and the setup script.
3. `livekit-softphone-test.md`: test the call path without a carrier.
4. `how-livekit-agents-work.md` and `sage-explained.md`: understand how the pieces fit.
5. `livekit-egress.md`: add recording when you need it.

## Notes

- These guides describe a prototype setup. Restrict SIP access and rotate passwords before using a real phone number.
- Commands are written for Windows PowerShell where noted. Linux commands are shown separately.
- Check LiveKit's current docs before relying on any command, since versions and method names change.
