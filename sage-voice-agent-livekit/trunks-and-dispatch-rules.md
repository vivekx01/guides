# Trunks and dispatch rules: a complete guide

This guide explains how phone calls reach a LiveKit room and an agent. It covers the two LiveKit objects that make that work, how to create them, how to set up the phone provider to send calls to your server, and what to check when something goes wrong.

It's written for anyone setting this up, not only for this project. Where a step depends on your provider or your server, placeholders in `<angle brackets>` show what to fill in.

| Placeholder | Meaning |
|---|---|
| `<sip-domain>` | The SIP subdomain that points at your server, such as `sip.example.com` |
| `<vps-ip>` | Your server's public IP address |
| `<phone-number>` | Your provider's phone number, in E.164 format, such as `+15551234567` |
| `<agent-name>` | The agent that should join each call, such as `sage` |

---

## Part 1: The big picture

When someone calls your number, the call passes through three stages:

```
Caller's phone
     │
     ▼
Provider (Twilio, Telnyx, Plivo, …)      ← owns the phone number, sends the call to you
     │  SIP
     ▼
LiveKit SIP service on your server        ← checks the call against a TRUNK
     │
     ▼
DISPATCH RULE                             ← decides which room the call goes into
     │
     ▼
LiveKit room  (call-xxxxx)                ← caller and agent meet here
     ▲
     │
Agent (Sage) joins the room
```

- The **provider** owns the phone number and delivers calls to your server. You configure it on the provider's website.
- The **trunk** is LiveKit's record of that number. It answers "should I accept a call to this number, and from whom?"
- The **dispatch rule** is LiveKit's routing decision. It answers "once a call is accepted, which room does it go into, and which agent joins?"

You configure the provider once, then the trunk and dispatch rule on the LiveKit side.

---

## Part 2: Glossary

| Term | Meaning |
|---|---|
| **SIP** | The protocol phone networks use to set up calls over the internet |
| **SIP trunk** | A connection between a phone provider and your server, used for calls |
| **Inbound trunk** | LiveKit's record of a phone number that accepts calls |
| **Dispatch rule** | LiveKit's routing rule, deciding which room a call enters and which agent joins |
| **Room** | A session where participants and agents exchange audio |
| **Room prefix** | The start of a room's name. LiveKit adds a unique ID after it |
| **Agent** | A program, such as Sage, that joins a room and talks |
| **Participant** | Anyone in a room: a caller, a person in a browser, or an agent |
| **E.164** | The international phone number format: `+` and country code, then digits, with no spaces |
| **Origination URI** | The address a provider uses to send calls to your server |
| **Allowed addresses** | IP addresses or ranges that are permitted to send calls to a trunk |
| **Credentials** | A username and password a provider sends so LiveKit can verify the call |
| **TTL** | How long a DNS record or token stays valid |

---

## Part 3: The inbound trunk

### What it does
The trunk tells LiveKit which phone numbers to accept calls for, and from where. A call to a number that has no matching trunk is rejected.

### Settings

| Setting | Example | Purpose |
|---|---|---|
| **Name** | `sage-inbound` | A label to recognize the trunk. Doesn't affect calls. |
| **Numbers** | `["+15551234567"]` | The phone numbers this trunk accepts. Must match the provider's number exactly. |
| **Allowed addresses** | `["203.0.113.10", "198.51.100.0/24"]` | IP addresses or CIDR ranges allowed to send calls. Empty means any address. |
| **Auth username and password** | (optional) | Credentials the provider must send. Use these when the provider supports them. |

### Allowed addresses
- Leaving this empty accepts calls from anyone who knows the number and the SIP address. That's fine for a short test, but not for a real number.
- Set it to the provider's signaling IP addresses for production. Check the provider's documentation for the current list, since it can change.
- You can use single IPs (`203.0.113.10`) or ranges (`198.51.100.0/24`).

### Credentials
Some providers, such as Twilio's TwiML route, send a username and password with each call. LiveKit checks them against the trunk. If you use credentials, the trunk's username and password must match exactly what the provider sends.

### Things to know
- Each number belongs to one inbound trunk. Creating a second trunk for the same number can cause conflicts.
- Changing allowed addresses or credentials applies to new calls. Calls already in progress are not affected.
- Deleting a trunk immediately stops calls to its number from being accepted.

---

## Part 4: The dispatch rule

### What it does
When a call is accepted, the dispatch rule decides:
1. **Which room** the caller goes into, and whether to create one.
2. **Which agent** joins that room.

### Rule types

| Type | Behavior | When to use |
|---|---|---|
| **Individual** | Each caller gets a new room. The room name starts with the room prefix, followed by a unique ID. | Voice agents, where every call should be separate. This is what we use. |
| **Direct** | Every caller goes into one fixed room. | Shared spaces, where callers can hear each other. Not suitable for voice agents. |
| **Callee** | A room per dialed number, named with a prefix. | When one service handles several numbers and you want rooms grouped by number. |

### Room prefix
The room prefix is the text at the start of each room's name. LiveKit adds a unique ID after it.

With prefix `call-`:
- Call 1 goes into `call-7f3a9c`
- Call 2 goes into `call-b21e04`

The prefix is only a label. It doesn't change how calls work. It helps you find call rooms in the room list and in logs. Any text works, as long as it's clear.

### Agent
The rule's **room configuration** names the agent to dispatch, such as `sage`. The agent name must exactly match the name the agent registers with. In this project, that's the `@server.rtc_session(agent_name="sage")` line in `sage.py`.

If the agent name doesn't match, the call still connects, but no agent joins.

### Trunk scope
A dispatch rule can apply to specific trunks, or to all trunks when no trunk is given. Our rule applies to all trunks, which is fine for one number.

### Duplicate rules
LiveKit won't create a second rule that matches the same trunk, number, and PIN as an existing one. The rule's name and room prefix don't count. To change a rule, delete the old one first.

---

## Part 5: Creating the trunk and rule

You can create them in three ways. They all make the same calls to the LiveKit server.

### Option A: the setup script

From the agent project folder:

```powershell
uv run python setup_sip.py dispatch                  # create the dispatch rule
uv run python setup_sip.py trunk <phone-number>      # create the inbound trunk
uv run python setup_sip.py list                      # check what exists
```

Set `SIP_ALLOWED_ADDRESSES` in `.env` before creating the trunk, to restrict which addresses can call.

The script doesn't set credentials yet. If your provider needs them, use Option B or C.

### Option B: the control app (browser)

1. Sign in to the control app.
2. Open **Dispatch rules**, enter a name and a room prefix, and click **Create rule**.
3. Open **Trunks**, enter a name, the phone number, and the allowed addresses, and click **Create trunk**.
4. Check the **Overview** page, which lists every trunk and rule from the LiveKit server.

Errors appear at the top of the page. For example, a duplicate rule shows LiveKit's message, and nothing is changed.

### Option C: the LiveKit API (code)

The Python SDK exposes the same calls:

```python
from livekit import api

async with api.LiveKitAPI(url, key, secret) as lk:
    trunk = await lk.sip.create_inbound_trunk(
        api.CreateSIPInboundTrunkRequest(
            trunk=api.SIPInboundTrunkInfo(
                name="sage-inbound",
                numbers=["+15551234567"],
                allowed_addresses=["203.0.113.10"],
            )
        )
    )

    rule = await lk.sip.create_dispatch_rule(
        api.CreateSIPDispatchRuleRequest(
            name="sage-calls",
            rule=api.SIPDispatchRule(
                dispatch_rule_individual=api.SIPDispatchRuleIndividual(room_prefix="call-"),
            ),
            room_config=api.RoomConfiguration(
                agents=[api.RoomAgentDispatch(agent_name="sage")],
            ),
        )
    )
```

Method names change between SDK versions. Check the installed package if a call fails.

### The same settings in the LiveKit CLI

If you use the `lk` CLI, the JSON looks like this:

```json
{
  "trunk": {
    "name": "sage-inbound",
    "numbers": ["+15551234567"],
    "allowedAddresses": ["203.0.113.10"]
  }
}
```

```json
{
  "dispatch_rule": {
    "rule": { "dispatchRuleIndividual": { "roomPrefix": "call-" } },
    "name": "sage-calls",
    "roomConfig": { "agents": [{ "agentName": "sage" }] }
  }
}
```

Create them with `lk sip inbound create <file>` and `lk sip dispatch create <file>`, and list them with `lk sip inbound list` and `lk sip dispatch list`.

---

## Part 6: Provider setup

The provider must send calls to your server. Set this up on the provider's website, using your `<sip-domain>`.

Before you start, make sure the SIP subdomain resolves to your server. See the DNS step in `livekit-sip-service.md`.

### Twilio

Twilio supports two routes. Pick one.

**Route 1: Elastic SIP trunk with IP-based access (recommended for this setup)**

1. In the Twilio console, go to **Elastic SIP Trunking** and create a trunk.
2. Under **Origination**, add an origination URI pointing at your server:
   `sip:<sip-domain>:5060`
   Use TCP or UDP transport, matching your port mapping.
3. Under **Phone numbers**, assign your Twilio number to the trunk.
4. In LiveKit, create the inbound trunk with the same number, and set **allowed addresses** to Twilio's signaling addresses for your region. Check Twilio's current published list.

Elastic trunks identify callers by IP address rather than username and password, which is why allowed addresses are the protection on the LiveKit side.

**Route 2: TwiML Bin with credentials**

This route sends a username and password with each call.

1. In Twilio, create a **TwiML Bin** with a `<Dial>` and `<Sip>` element like this:
   ```xml
   <Response>
     <Dial>
       <Sip username="<username>" password="<password>">
         sip:<phone-number>@<sip-domain>;transport=tcp
       </Sip>
     </Dial>
   </Response>
   ```
2. In **Phone Numbers**, set the number's voice configuration to use this TwiML Bin.
3. In LiveKit, the inbound trunk must have the same username and password. **Our tooling doesn't set these yet.** Use the SDK's `auth_username` and `auth_password` fields, or add them to the control app.

Twilio's TwiML route doesn't support outbound calls or call transfers. Use Route 1 if you need them.

Twilio's own documentation is the authority for these settings, since the console changes over time. The LiveKit Twilio guide uses LiveKit Cloud's SIP host, so replace it with your own `<sip-domain>`.

### Telnyx

1. In the Telnyx portal, create a **SIP connection** with the type set to **FQDN**.
2. Set the destination to your `<sip-domain>` on port 5060.
3. Choose **Credentials** for authentication if you want credential-based checks. Otherwise, use IP-based checks as with Twilio.
4. Assign your Telnyx number to the connection.
5. In LiveKit, create the inbound trunk with the same number, and set allowed addresses to Telnyx's signaling addresses if you use IP checks.

If you use credentials, they must match the trunk's username and password, which needs the same SDK fields as Twilio's TwiML route.

### Plivo and Wavix

LiveKit has quickstart guides for both. I haven't verified their steps in this guide, so follow LiveKit's current Plivo and Wavix pages for the provider side. The LiveKit side is the same as above: an inbound trunk with the number, and a dispatch rule.

### Indian numbers

Indian phone numbers usually need the provider to register your business first, so a foreign provider may not sell them to individuals. Check the provider's country coverage before you buy a number. For testing, the softphone route in `livekit-softphone-test.md` doesn't need a provider or a number.

---

## Part 7: Test end to end

1. **Check the server side.** Run `setup_sip.py list`, or open the control app's Overview. Both the trunk and the dispatch rule should appear.
2. **Start the agent.** Run `uv run python sage.py dev` locally, or confirm the VPS copy shows `registered worker`.
3. **Place a call** from a phone to the provider number, or from a softphone to the SIP address.
4. **Watch the SIP service logs.** You should see the call accepted and a participant join a room.
5. **Watch the agent.** It should join the new `call-…` room.
6. **Confirm the audio.** Speak, and check that the agent replies.

---

## Part 8: Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| Creating a rule fails with "already exists" | A rule with the same trunk, number, and PIN exists | Delete the existing rule first, then create the new one |
| The call doesn't connect at all | No trunk for the number, the number doesn't match exactly, or port 5060 is blocked | Check the trunk's number against the provider's number, and the firewall |
| The call connects but no agent joins | No dispatch rule, or the agent name doesn't match | Check the rule's agent name is `sage`, and the agent is running |
| The call is rejected with 403 | Allowed addresses don't include the provider's address, or credentials don't match | Add the provider's addresses, or check the credentials on both sides |
| The provider says the call failed | Provider can't reach `<sip-domain>` | Check the DNS record, and that the SIP service is running on 5060 |
| Calls connect but there's no audio | The media ports (10000–10100 UDP) are blocked | Check the port mapping and the firewall |
| The agent joins but ignores the caller | The agent is linked to a different participant | Check the logs for which participant the agent linked to |
| Changes don't take effect | The call started before the change | Place a new call |
| The control app shows an error on the Overview | The control app can't reach the LiveKit server | Check `LIVEKIT_URL` and the key pair in the control app |

---

## Part 9: Security

- **Restrict allowed addresses** to the provider's signaling addresses before using a real number. Without this, anyone who knows the number and the SIP address can reach your agent, and every call costs you provider and AI usage.
- **Use credentials** if the provider supports them, in addition to IP restrictions.
- **Delete test trunks** when you finish testing, so the test number can't be used by anyone else.
- **Keep API keys and credentials** out of documents and chats. Store them in a password manager.

---

## Part 10: Known gaps in our setup

- **The setup script and control app don't set credentials** on the trunk. Route 2 for Twilio, and any provider that needs credentials, requires adding those fields.
- **The control app can't edit trunks yet.** To change a trunk, delete it and create a new one.
- **Provider steps for Twilio and Telnyx** come from their public documentation and LiveKit's guides. Check the current screens, since providers change them.
- **Provider IP addresses** for allowed addresses must be confirmed against each provider's current list.

---

## Quick reference

| Item | Value |
|---|---|
| SIP address for providers | `<sip-domain>`, port 5060 |
| Inbound trunk | Number, optional allowed addresses, optional credentials |
| Dispatch rule type | Individual |
| Room prefix | `call-` (any text works) |
| Agent name | `sage` |
| Script commands | `setup_sip.py dispatch`, `trunk <number>`, `list` |
| Duplicate rule | Delete the old rule, then create the new one |
