# Testing the Sage voice agent with a softphone

This guide shows how to test the full call path without a phone carrier. A softphone on your computer dials the LiveKit SIP service directly, which places the call in a room and sends Sage to it.

This was verified on our setup: a call from Linphone was accepted, joined a room, and stayed connected.

Placeholders in `<angle brackets>` are values you supply:

| Placeholder | Meaning |
|---|---|
| `<sip-domain>` | The SIP subdomain, such as `sip.example.com` |
| `<test-number>` | The made-up number used for the test trunk, such as `+919999900000` |
| `<caller-ip>` | Your computer's public IP, for the allowed addresses setting |

---

## 1. What this test covers

```
Softphone (your PC)
   │  SIP call to sip:<test-number>@<sip-domain>
   ▼
LiveKit SIP service (VPS, port 5060)
   │  matches the inbound trunk by number
   │  applies the dispatch rule, creates a call-… room
   ▼
LiveKit room ◀── Sage joins (agent dispatched by the rule)
```

It doesn't cover the phone carrier, so it doesn't test:
- phone numbers from a provider
- carrier billing
- call routing from the public phone network

Those depend on a SIP trunking provider, covered in `livekit-sip-service.md`.

---

## 2. Prerequisites

- **LiveKit server and SIP service running** on the VPS. See `livekit-self-host-coolify.md` and `livekit-sip-service.md`.
- **Sage running** on your computer with `uv run python sage.py dev`. Wait for the `registered worker` line.
- **An inbound trunk and a dispatch rule** created with `setup_sip.py`.
- **A softphone** installed. This guide uses Linphone, which is free.

---

## 3. Set up the trunk and dispatch rule

Run these from the agent project folder, in this order:

```powershell
uv run python setup_sip.py dispatch                 # create the dispatch rule
uv run python setup_sip.py trunk <test-number>      # create the inbound trunk
uv run python setup_sip.py list                     # confirm both exist
```

Expected output of `list`:

```
Inbound trunks:
  ST_xxxxxxxx  sage-inbound  numbers=['<test-number>']
Dispatch rules:
  SDR_xxxxxxxx  sage-calls  agents=['sage']
```

**Both are required.** The trunk accepts the call. The dispatch rule decides which room the call goes into and sends Sage there. Without the dispatch rule, the call connects but Sage won't join.

The `dispatch` command is separate from `trunk`. Running `trunk` alone doesn't create the rule.

---

## 4. Allow your computer's address (optional, but recommended)

By default, the trunk accepts calls from any address. For the test, that's acceptable. For anything longer, restrict it to your address:

1. Find your public IP (search "what is my IP").
2. Add it to `SIP_ALLOWED_ADDRESSES` in `.env`:
   ```
   SIP_ALLOWED_ADDRESSES=<caller-ip>
   ```
3. Recreate the trunk, since allowed addresses are set when the trunk is created.

Remove the test trunk when you're done, so the number can't be used by anyone else.

---

## 5. Install and set up Linphone

1. Download Linphone from its official website, and install it.
2. Open it and skip account creation. You're calling a server address, not a registered user.
3. Open the settings and check that the transport is set to **TCP** or **UDP**. Both work with the port mapping, since port 5060 is open on both.

---

## 6. Place the call

1. Make sure Sage is running in its terminal.
2. In Linphone's address bar, enter:
   ```
   sip:<test-number>@<sip-domain>
   ```
   For example: `sip:+919999900000@sip.example.com`
3. Press the call button.

---

## 7. What a successful call looks like

**SIP service logs** (on the VPS):

```
SIP participant joined room
published track
Accepting the call
track subscribed
ACK from remote
call statistics
```

The `call statistics` lines repeat once a minute while the call is active. That's a good sign that media is flowing.

**Sage's terminal:**
- A job for a new `call-…` room
- Sage joining the room
- Speech being recognized and replies being spoken

**What you should hear:** Sage's greeting, then replies when you speak.

To check the room from the agent project folder, run `setup_sip.py list` again. The room itself appears in LiveKit while the call is active.

---

## 8. Things you may see in the logs

These are harmless during the test:

- **`Inbound SIP request not handled` with method `SUBSCRIBE`:** Linphone sends an extra request for its conference feature. The SIP service ignores it.
- **`timestamp gap` or `silence_filler`:** the SIP service fills short gaps in audio, which can happen when you're silent.

---

## 9. Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| Call doesn't connect at all | Wrong address, port 5060 blocked, or the SIP service isn't running | Check the address, transport, and that the SIP container is up. Check the port with `ss -lntu` on the VPS. |
| Call connects but no Sage | No dispatch rule, or Sage isn't running | Run `setup_sip.py list`, and confirm `sage.py dev` is running |
| Sage joins but doesn't speak | Agent or provider key problem | Check Sage's terminal for errors from Deepgram, OpenRouter, or Fish |
| `Forbidden` or `403` | Trunk rejected the call, often from the allowed addresses | Check `SIP_ALLOWED_ADDRESSES`, or recreate the trunk without it |
| Call drops after a few seconds | Media ports blocked | Check the RTP range (10000–10100 UDP) in the port mappings and firewall |
| No audio in either direction | Media not reaching the server | Check the RTP port mapping, and the `use_external_ip` setting |

---

## 10. After the test

1. **Remove the test trunk** if you won't use it again. The trunk and dispatch rule need to be deleted through the LiveKit API, since `setup_sip.py` doesn't have a delete command yet.
2. **Restrict access** before using a real number. See `livekit-sip-service.md`, section 11.
3. **Move Sage to the VPS** when you're ready, so it stays available without your computer.

---

## 11. Quick reference

| Item | Value |
|---|---|
| Dial address | `sip:<test-number>@<sip-domain>` |
| Transport | TCP or UDP, port 5060 |
| Create dispatch rule | `uv run python setup_sip.py dispatch` |
| Create trunk | `uv run python setup_sip.py trunk <test-number>` |
| Check both | `uv run python setup_sip.py list` |
| Start Sage | `uv run python sage.py dev` |
| Agent name | `sage` |
| Room prefix | `call-` |
