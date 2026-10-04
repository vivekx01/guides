# Sage stack: the agent, the control panel, and Redis

This guide covers the three parts we wrote for this project and run on the VPS: the Sage agent, the LiveKit control panel, and the Redis instance that LiveKit and the SIP service share. It's written so you can rebuild the stack from scratch on a new server.

Placeholders in `<angle brackets>` stand for your own values. Keep real keys, passwords, and tokens in Coolify's environment settings or a password manager, never in this file or in the repo.

| Placeholder | Meaning |
|---|---|
| `<your-domain>` | Your base domain, such as `example.com` |
| `<control-domain>` | Subdomain for the control panel, such as `control.example.com` |
| `<redis-address>` | Redis host and port inside Coolify, such as `<redis-container>:6379` |
| `<redis-password>` | Redis password |
| `<redis-db>` | A Redis database number not used by any other app, such as `3` |
| `<internal-token>` | Shared secret that Sage uses to read its settings |
| `<session-secret>` | Secret that signs control panel login sessions |

---

## Part 1: How the three parts fit together

```
            ┌─────────────────────┐
            │  Control panel      │  trunks, dispatch rules, per-agent settings
            │  (Coolify, /control)│
            └──────────┬──────────┘
                       │ reads and writes via LiveKit API
                       │ serves each agent's settings over HTTPS
┌──────────┐   ┌───────▼──────────┐   ┌─────────────────┐
│  Sage    │──▶│ LiveKit server   │◀──│ LiveKit SIP     │
│ (agent)  │   │ (Coolify)        │   │ (Coolify)       │
└──────────┘   └───────┬──────────┘   └────────┬────────┘
                       │                       │
                       └────────┬──────────────┘
                                ▼
                        Redis (shared state)
```

- **Sage** is the voice agent. It registers with LiveKit, joins rooms when dispatched, and reads its settings from the control panel at the start of each call.
- **The control panel** is a web app for managing phone routing and settings. It talks to LiveKit's API with your key pair, so the browser never sees the keys.
- **Redis** holds shared state for the LiveKit server and the SIP service. Both must point at the same instance and database.

---

## Part 2: Redis

LiveKit's server and the SIP service both require Redis in this setup. Without it, the SIP service won't start.

### Choose a database

Redis has numbered databases. Use one that no other app uses. Check which apps already connect to your Redis, and their database numbers, before picking one. We used database `3` because the other apps used `0` and `2`.

Using a separate Redis container avoids sharing a failure point with other apps. A shared instance works, but a restart or a flush affects everything that uses it.

### Find the connection details

In Coolify, open the Redis resource and note:
- its internal container name, which is the address other services use
- its password, which Coolify stores as `REDIS_PASSWORD`
- its username, which is `default` for the standard image

Verify the password works before you configure anything:

```bash
docker exec <redis-container> sh -c 'redis-cli --no-auth-warning --user default -a "$REDIS_PASSWORD" PING'
```

It should print `PONG`. Use the container's own environment variable, so the password doesn't appear in the command line.

### Configure LiveKit

Add a `redis` section to the LiveKit server's `LIVEKIT_CONFIG` environment variable, next to its RTC settings:

```
{rtc: {tcp_port: 7881, port_range_start: 50000, port_range_end: 50100, use_external_ip: true}, redis: {address: <redis-address>, username: default, password: <redis-password>, db: <redis-db>}}
```

### Configure the SIP service

The SIP service's `SIP_CONFIG_BODY` variable needs the same Redis details, in its own config:

```
{api_key: <api-key>, api_secret: <api-secret>, ws_url: wss://<livekit-domain>, redis: {address: <redis-address>, username: default, password: <redis-password>, db: <redis-db>}, sip_port: 5060, rtp_port: 10000-10100, use_external_ip: true, logging: {level: info}}
```

### Rules for these variables

- **No quotes around the password.** Coolify saved quoted values with literal backslashes, so the password Redis received was wrong. Passwords here use letters and numbers, so quotes aren't needed.
- **A space after every colon.** Without it, the value parses incorrectly.
- **One line.** Keep each variable on a single line.

### Verify

- **LiveKit logs** should show `connecting to redis` followed by `starting LiveKit server`, with no `WRONGPASS` error. The `single-node routing` line should be gone.
- **SIP logs** should show `connecting to redis`, then `sip signaling listening on` for port 5060 over both UDP and TCP.
- **A `WRONGPASS invalid username-password pair` error** means the password in the variable doesn't match the one Redis is using. Check the password, then remove any quotes or backslashes and redeploy.

### Rotating the password

If a password has been shared, change it in the Redis resource, then update every service that uses it, including LiveKit, the SIP service, and any other app that connects. Redeploy each one, and check its logs.

---

## Part 3: The control panel

The control panel is a small web app in the `control/` folder of the Sage repo. It runs in Coolify from the same GitHub repository.

### Deploy in Coolify

1. **Create a resource** from the GitHub repo, with the **Dockerfile** build pack and base directory `/control`.
2. **Set the domain** to `<control-domain>` with HTTPS, and set the port to `8000`.
3. **Add a persistent volume** mounted at `/data`. The settings database lives there, so it survives redeploys.
4. **Set the environment variables** listed below.
5. **Deploy**, then check the logs for the server starting on port 8000.

If Coolify reports "no available server" when you deploy, the resource has no server or destination set. Set both: the server should be the local one, and the destination the default Docker network.

### Environment variables

| Variable | Purpose |
|---|---|
| `CONTROL_USERNAME` | The sign-in username |
| `CONTROL_PASSWORD` | The sign-in password |
| `SESSION_SECRET` | Signs login cookies. Use a long random value. |
| `INTERNAL_TOKEN` | The token agents send to read their settings. Use the same value in Sage. |
| `LIVEKIT_URL` | `wss://<livekit-domain>` |
| `LIVEKIT_API_KEY` | The LiveKit API key |
| `LIVEKIT_API_SECRET` | The LiveKit API secret |

The Dockerfile sets `DB_PATH` to `/data/control.db` and `HTTPS_ONLY` to `true`, so you don't need to add them.

To generate `SESSION_SECRET` and `INTERNAL_TOKEN`, use a random 48-character value. In PowerShell:

```powershell
-join ((48..57) + (65..90) + (97..122) | Get-Random -Count 48 | ForEach-Object { [char]$_ })
```

### Pages

- **Overview:** the trunks and dispatch rules that exist on the LiveKit server.
- **Trunks:** create, edit, and delete inbound trunks. Each trunk has a name, a phone number in E.164 format, and optional allowed addresses.
- **Dispatch rules:** create, edit, and delete rules. Each rule has a name, a room prefix, the agents to dispatch, and the trunks it applies to. Leave all trunks unticked to apply the rule to every trunk.
- **Agents:** lists every agent that has settings, and creates new ones. Each agent has its own settings page at `/agents/<name>/settings`.

### Agent names

An agent name must match three things exactly:
1. The `agent_name` in the agent's code.
2. The agent name in the dispatch rule.
3. The name on the Agents page.

To find an agent's name, check its startup log for `registered worker` with the `agent_name` value.

### Internal API

Agents read their settings from:

```
GET https://<control-domain>/api/internal/agents/<agent-name>/settings
Authorization: Bearer <internal-token>
```

If an agent has no saved settings, the panel returns the defaults and logs a warning, so calls still work. A request without the right token gets `401`.

### Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| Overview shows "could not reach the LiveKit server" | `LIVEKIT_URL` or the key pair is wrong | Check the three LiveKit variables and redeploy |
| Creating a rule fails with "already exists" | Another rule covers the same trunk, number, and PIN | Change the trunks, or delete the existing rule first |
| Changes don't reach Sage | Sage's `CONTROL_URL` or `INTERNAL_TOKEN` is wrong | Check both values in Sage's settings |
| Login fails | Wrong username or password, or cookies blocked | Check the variables, and use HTTPS |

---

## Part 4: The Sage agent

Sage is the voice agent. Its code is in the root of the Sage repo, and its Dockerfile builds the image Coolify runs.

### Deploy in Coolify

1. **Create a resource** from the GitHub repo, with the **Dockerfile** build pack and base directory `/`.
2. **Leave the domain and port mappings empty.** Sage only makes outgoing connections.
3. **Set the environment variables** listed below.
4. **Deploy.** The build downloads the Silero and turn-detector model files, so the first build takes a few minutes.
5. **Check the logs** for `registered worker` with `agent_name` set to `sage`.

Stop any local Sage process before testing the server copy. Two workers registered under the same name can both receive calls.

### Environment variables

| Variable | Purpose |
|---|---|
| `LIVEKIT_URL` | `wss://<livekit-domain>` |
| `LIVEKIT_API_KEY` | The LiveKit API key |
| `LIVEKIT_API_SECRET` | The LiveKit API secret |
| `DEEPGRAM_API_KEY` | Speech-to-text |
| `OPENROUTER_API_KEY` | The language model |
| `FISH_API_KEY` | Text-to-speech |
| `CONTROL_URL` | `https://<control-domain>`, with no path. Sage reads its settings from here. |
| `INTERNAL_TOKEN` | The same value as the control panel's `INTERNAL_TOKEN` |

If `CONTROL_URL` or `INTERNAL_TOKEN` is missing, Sage uses its built-in defaults and logs that it's doing so.

### Settings

Sage's settings are the greeting, instructions, language model, speech-to-text model, and voice. They're stored in the control panel under the agent name `sage`. Changes apply to the next call, so you don't need to redeploy.

### Running locally

For a local test, copy `.env.example` to `.env`, fill in the values, and run one of:

```powershell
uv run python sage.py console   # talk through your microphone, no LiveKit needed
uv run python sage.py dev       # connect to the LiveKit server for testing
uv run python sage.py start     # production mode, the same as Coolify runs
```

### Adding another agent

1. Copy the agent code, and change the agent name in the decorator, for example `@server.rtc_session(agent_name="sage2")`. Also change `AGENT_NAME` so the settings request uses the new name.
2. Deploy it as its own Coolify resource, with its own environment variables.
3. Create the agent on the control panel's Agents page, and set its settings.
4. Use the same name in a dispatch rule.

Each agent is its own program. Two agents can't share one worker.

### Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| No `registered worker` line | LiveKit URL or key pair is wrong | Check the three LiveKit variables |
| Worker registers, calls don't reach it | No dispatch rule names this agent | Check the rule's agent name |
| Agent uses default greeting or instructions | Control panel settings unreachable, or no saved settings | Check `CONTROL_URL`, `INTERNAL_TOKEN`, and the agent's Agents page entry |
| Agent joins but doesn't speak | A provider key is wrong | Check the Deepgram, OpenRouter, and Fish keys in its logs |
| Memory climbs during calls | Several calls running at once | Check `free -h` and the container's memory in Coolify |

---

## Part 5: Quick reference

| Item | Value |
|---|---|
| Sage repo | The root of the Sage repo, built by its Dockerfile |
| Control panel | `control/` in the Sage repo, base directory `/control`, port 8000, volume `/data` |
| Redis | `<redis-address>`, user `default`, database `<redis-db>` |
| Settings endpoint | `/api/internal/agents/<agent-name>/settings` |
| Agent name for Sage | `sage` |
| Startup line to look for | `registered worker` with the agent's name |
