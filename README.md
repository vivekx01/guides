# Guides

Step-by-step guides for building and running voice agents on your own infrastructure.

Each topic has its own folder, with a README that lists its guides in reading order.

---

## Topics

### [Sage voice agent with self-hosted LiveKit](sage-voice-agent-livekit/)

Run a phone-capable AI voice agent on your own server: LiveKit for rooms, a SIP service for phone calls, and an agent that listens, thinks, and speaks.

| Guide | What it covers |
|---|---|
| [Sage, explained](sage-voice-agent-livekit/sage-explained.md) | Start here. The whole system in plain language, with code examples. |
| [How LiveKit agents work](sage-voice-agent-livekit/how-livekit-agents-work.md) | How agents register, how tokens let people join rooms, and how dispatch brings an agent in. |
| [Self-hosting LiveKit on Coolify](sage-voice-agent-livekit/livekit-self-host-coolify.md) | Installing the LiveKit server on a VPS: swap, keys, ports, and verification. |
| [LiveKit SIP service](sage-voice-agent-livekit/livekit-sip-service.md) | Running the SIP service: the DNS record, the trunk and dispatch rule, and connecting a phone provider. |
| [Trunks and dispatch rules](sage-voice-agent-livekit/trunks-and-dispatch-rules.md) | A complete guide to inbound trunks and dispatch rules, with setup options and provider configuration for Twilio and Telnyx. |
| [Softphone test](sage-voice-agent-livekit/livekit-softphone-test.md) | Testing the full call path with a free softphone, without a phone carrier. |
| [Egress](sage-voice-agent-livekit/livekit-egress.md) | Recording rooms and tracks with LiveKit Egress. |

**Suggested reading order**

1. Self-hosting LiveKit on Coolify
2. LiveKit SIP service
3. Trunks and dispatch rules
4. Softphone test
5. How LiveKit agents work, and Sage, explained
6. Egress, when you need recordings

---

## Notes

- Guides use placeholders in `<angle brackets>` for IPs, domains, keys, and passwords. Never put real values in these files.
- Commands are shown for Windows PowerShell where noted, with Linux equivalents alongside.
- Check each provider's and LiveKit's current documentation before relying on a step, since consoles and APIs change.
