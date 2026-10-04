# Setting up the LiveKit SIP service on a VPS with Coolify

This guide explains how to run the LiveKit SIP service, which lets phone calls reach LiveKit rooms. It's written so you can repeat the setup on any VPS running Coolify, or adapt it to another Docker host.

Placeholders in `<angle brackets>` are values you supply:

| Placeholder | Meaning |
|---|---|
| `<vps-ip>` | The VPS's public IP address |
| `<your-domain>` | The domain for your LiveKit server, such as `example.com` |
| `<sip-domain>` | The subdomain for SIP, such as `sip.example.com` |
| `<api-key>` / `<api-secret>` | The LiveKit key pair, from `LIVEKIT_KEYS` |
| `<redis-address>` | Host and port of Redis, such as `<redis-container>:6379` |
| `<redis-password>` | Redis password. Keep it only in Coolify and `.env`, never in documents or chats |
| `<redis-db>` | A Redis database number no other app uses, such as `3` |
| `<phone-number>` | Your provider's number in E.164 format, such as `+15551234567` |

---

## 1. What the SIP service does

- It receives phone calls from a provider such as Twilio, on port 5060.
- It checks each call against an **inbound trunk**, which holds the phone number and any allowed provider addresses.
- It uses a **dispatch rule** to choose a room, creating the room if it doesn't exist.
- It connects the caller to that room as a SIP participant, converting phone audio into the same format browsers use.

Agents such as Sage join the room through normal dispatch, the same way browser participants do.

The SIP service doesn't create trunks or dispatch rules. You create those through the LiveKit server's API (or the CLI), and the SIP service reads them when a call arrives.

---

## 2. Before you start

You need:

- **A working LiveKit server** with a public domain and HTTPS. See `livekit-self-host-coolify.md`.
- **Redis**, reachable from both the LiveKit server and the SIP service, using a database number no other app uses.
- **A static public IP** for the VPS.
- **A SIP trunking provider** with a phone number, such as Twilio.
- **DNS control** for your domain, so you can create the SIP subdomain (section 4).

Redis is required. The SIP service won't start without a working connection to it.

---

## 3. Create the SIP resource in Coolify

Add a **Docker Image** resource with these settings:

| Setting | Value |
|---|---|
| Image | `livekit/sip` |
| Tag | `latest` for now. Pin a version later. |
| Domain | Leave empty. SIP isn't web traffic, so it doesn't use a Coolify domain. |
| Ports exposes | Leave empty |
| Port mappings | `5060:5060,5060:5060/udp,10000-10100:10000-10100/udp` |
| Network | Coolify's default network. Don't use host networking, because the SIP service needs to resolve the Redis container by name. |

The port mappings publish:
- **5060 over TCP and UDP** for SIP signaling
- **10000–10100 over UDP** for call audio. Size this to your expected number of simultaneous calls, since each port is published separately.

**Environment variable:** `SIP_CONFIG_BODY`, entered as one line:

```
{api_key: <api-key>, api_secret: <api-secret>, ws_url: wss://<your-domain>, redis: {address: <redis-address>, username: default, password: <redis-password>, db: <redis-db>}, sip_port: 5060, rtp_port: 10000-10100, use_external_ip: true, logging: {level: info}}
```

| Setting | Purpose |
|---|---|
| `api_key`, `api_secret` | The LiveKit key pair. Must match the LiveKit server's `LIVEKIT_KEYS`. |
| `ws_url` | The LiveKit server's WebSocket address. The public `wss://` URL works. |
| `redis` | Redis connection. Same address, user, and database as the LiveKit server. |
| `sip_port` | SIP signaling port. The default is 5060. |
| `rtp_port` | Call audio port range. Must match the port mapping. |
| `use_external_ip` | Advertises the public IP instead of the private Docker address. Required on a VPS. |
| `logging.level` | `info` normally, `debug` when troubleshooting |

**Formatting rules:**
- **No quotes around the password.** Coolify can save quotes with literal backslashes, which makes the password wrong.
- **A space after every colon.** Without it, the value parses incorrectly.
- **Enter the value only,** starting with `{`, on one line.

---

## 4. Create the SIP subdomain in DNS

The phone provider needs a hostname to send calls to. Create a DNS record for it.

| Field | Value |
|---|---|
| **Type** | `A` |
| **Name** | `sip` (your DNS provider adds the domain automatically) |
| **Points to / IPv4** | `<vps-ip>` |
| **TTL** | Default, such as 3600 seconds |

If your DNS provider offers a proxy or CDN option for this record, **turn it off**. Proxies don't pass SIP traffic.

Check the record from your computer:

```powershell
Resolve-DnsName <sip-domain> -Type A
```

It should return `<vps-ip>`. DNS changes can take a few minutes.

Use `<sip-domain>` as the destination in the provider's trunk settings, not the IP.

---

## 5. Configure the LiveKit server for Redis

The LiveKit server needs the same Redis settings. Add a `redis` section to its `LIVEKIT_CONFIG` variable, next to the RTC settings:

```
{rtc: {tcp_port: 7881, port_range_start: 50000, port_range_end: 50100, use_external_ip: true}, redis: {address: <redis-address>, username: default, password: <redis-password>, db: <redis-db>}}
```

Once Redis is configured, the server starts in distributed mode. The `single-node routing` line should no longer appear at startup.

Redeploy the LiveKit server first, then the SIP service.

---

## 6. Open the firewall

Allow these ports on the VPS, and on the provider's firewall panel if it has one:

| Port | Protocol | Purpose |
|---|---|---|
| 5060 | TCP and UDP | SIP signaling |
| 10000–10100 | UDP | Call audio |
| 7881 | TCP | LiveKit media fallback |
| 50000–50100 | UDP | LiveKit WebRTC media |

Check the current state first:

```bash
ufw status verbose        # if ufw is installed
iptables -S INPUT         # shows the default policy
```

---

## 7. Deploy and verify the SIP service

Deploy and watch the SIP logs. A working start looks like this:

```
connecting to redis
found external IP via STUN ... "externalIP": "<vps-ip>"
sip signaling listening on ... "port": 5060, "proto": "udp"
sip signaling listening on ... "port": 5060, "proto": "tcp"
```

On the VPS:

```bash
ss -lntu | grep -E ':5060|:7881'
docker ps --format '{{.Names}}\t{{.Status}}' | grep -iE 'livekit|sip'
free -h
dmesg -T | grep -i 'out of memory'
```

The LiveKit server should show:

```
connecting to redis
starting LiveKit server ... "rtc.portICERange": [50000, 50100]
```

**What the logs don't prove:** a healthy SIP start doesn't show a connection to the LiveKit server. That's confirmed only when a trunk exists and a call arrives.

---

## 8. Create the dispatch rule and inbound trunk

Use the helper script `setup_sip.py` in the agent project. It calls the LiveKit server's API with the key pair from `.env`, so no secrets go into the script.

Add these optional settings to `.env`:

```
SIP_ALLOWED_ADDRESSES=<comma-separated provider IPs or ranges>
```

Then run from the agent project folder:

```powershell
uv run python setup_sip.py list                    # show existing trunks and rules
uv run python setup_sip.py dispatch                # create the dispatch rule
uv run python setup_sip.py trunk <phone-number>    # create the inbound trunk
```

### What each one creates

**Dispatch rule (`dispatch`):**
- Each incoming call goes into its own room, named with the prefix `call-`.
- The `sage` agent is dispatched into each room, so it joins when a call arrives.
- Its settings: `dispatchRuleIndividual` with `room_prefix: call-`, and `room_config.agents` with `agent_name: sage`.

**Inbound trunk (`trunk`):**
- Accepts calls to `<phone-number>`.
- Limits calls to `SIP_ALLOWED_ADDRESSES` if set. The script warns if it isn't set.

### Notes on the script

- The dispatch rule doesn't need a phone number, so you can create it before the provider account is ready.
- The LiveKit docs say that restricting trunks by address may need a support request for your project. Confirm this before relying on it.
- The script uses the current SDK method names (`list_inbound_trunk`, `list_dispatch_rule`). The older names are deprecated and produce warnings.
- Inbound trunks don't have username and password fields set by this script, so access control depends on the allowed addresses.

---

### Doing the same steps in the control app

Once the control app is deployed (see the repo README), you can do the same steps from a browser instead of the script. Sign in with the control app's username and password.

**Create the dispatch rule**
1. Open **Dispatch rules** in the top menu.
2. Under **Add a rule**, enter a name such as `sage-calls`, and keep the room prefix as `call-`.
3. Click **Create rule**.
4. Check the **Existing rules** table. It should show the rule with `sage` in the Agents column.

**Create the inbound trunk**
1. Open **Trunks** in the top menu.
2. Under **Add a trunk**, enter a name such as `sage-inbound` and the phone number in E.164 format, for example `+15551234567`.
3. In **Allowed addresses**, enter your provider's addresses, separated by commas. Leave it empty only for testing.
4. Click **Create trunk**.

**Check the result**
- The **Overview** page lists every trunk and dispatch rule from the LiveKit server. Both should appear there.
- If the page shows "could not reach the LiveKit server," check the control app's `LIVEKIT_URL` and key pair.

**Deleting**
- On the **Trunks** or **Dispatch rules** page, click **Delete** on the row and confirm. Delete the test trunk when you're done testing.

The UI and the script make the same calls to the LiveKit server, so either works. Use whichever you prefer.

## 9. Set up the provider side

These steps depend on your provider's account being active. For Twilio:

1. **Buy a US number** in the Twilio console.
2. **Create an Elastic SIP trunk.**
3. **Set the origination URI** to `sip:<sip-domain>:5060`, using TCP or UDP transport.
4. **Attach the number** to the trunk.
5. **Note the provider's signaling IP addresses** for `SIP_ALLOWED_ADDRESSES` if they're published. Otherwise, check the provider's docs for the current ranges.

Twilio's Elastic trunks check calls by IP. Its username and password option uses a different setup (TwiML Bin), which we haven't tested with this server, so stick with IP checks unless you confirm otherwise.

---

## 10. Test with one call

1. Start the agent: `uv run python sage.py dev`.
2. Confirm the dispatch rule and trunk exist with `setup_sip.py list`.
3. Call the phone number from a different phone.
4. Watch the logs:
   - **SIP service:** the incoming call and the room it's placed in.
   - **Agent:** a job for the new `call-…` room, then the agent joining.
5. Check that the agent greets the caller and responds.

If the call doesn't connect, check the trunk's allowed addresses, the origination URI, and that port 5060 is open.

---

## 11. Security

- **Restrict calls to known provider addresses** using `SIP_ALLOWED_ADDRESSES`. Otherwise scanners and spam calls can reach the server.
- **Rotate Redis passwords** if they've been shared, and update every place they're stored: `LIVEKIT_CONFIG`, `SIP_CONFIG_BODY`, and any app connections.
- **Keep the key pair private.** Anyone with the API secret can create room tokens.
- **Keep credentials files private,** including SSH logins and API tokens.

---

## 12. Common problems

| Symptom | Likely cause | Fix |
|---|---|---|
| `WRONGPASS invalid username-password pair` in LiveKit and SIP logs | Wrong Redis password, or quotes saved with literal backslashes | Check the password, remove the quotes, and redeploy |
| SIP container keeps restarting | Can't connect to Redis, so it exits | Fix the Redis settings and redeploy |
| `Missing required permissions: deploy` from the Coolify API | Token has write access but not deploy | Create a token with deploy permission, or redeploy from the dashboard |
| SIP can't reach Redis by name | Container is on host networking | Use the default network with port mappings |
| Calls don't reach the agent | No dispatch rule, or the rule's agent name doesn't match | Run `setup_sip.py list`, and check `agent_name` is `sage` |
| Provider can't reach `sip.<domain>` | DNS record missing, wrong, or proxied | Check the A record with `Resolve-DnsName`, and turn off any proxy |
| Calls connect but no audio | Media ports blocked or not published | Check the RTP port mapping and firewall |
| Deprecation warnings from the script | Old method names | Use the current names in `setup_sip.py` |

---

## 13. Quick reference

| Item | Value |
|---|---|
| SIP domain | `<sip-domain>` (A record to `<vps-ip>`) |
| SIP signaling | Port 5060, TCP and UDP |
| SIP media | Range in `rtp_port`, UDP |
| WebRTC media | Range in the LiveKit RTC settings, UDP, plus TCP 7881 |
| Redis | `<redis-address>`, user `default`, database `<redis-db>` |
| Server URL | `wss://<your-domain>` |
| Setup script | `sage-voice-agent/setup_sip.py` with `list`, `dispatch`, and `trunk` commands |
| Agent name | `sage` |
| Room prefix | `call-` |
