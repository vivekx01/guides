# Self-hosting the LiveKit server on a VPS with Coolify

This guide explains how to run a LiveKit server on your own VPS through Coolify. It's written so you can repeat the setup on any VPS that runs Coolify, or adapt it to another Docker host.

Placeholders in `<angle brackets>` are values you supply:

| Placeholder | Meaning |
|---|---|
| `<vps-ip>` | The VPS's public IP address |
| `<your-domain>` | The subdomain for LiveKit, such as `livekit.example.com`, pointing at `<vps-ip>` |
| `<api-key>` / `<api-secret>` | A key pair you generate yourself (section 4) |

Store the key pair and any passwords in a password manager, not in documents or chats.

---

## 1. What you end up with

- **LiveKit server** running as a Coolify resource, reachable at `https://<your-domain>`. Coolify's proxy handles HTTPS and forwards signaling on port 7880.
- **TCP 7881** and a **UDP range** published directly on the VPS for media.
- **One key pair** used by your agents, the SIP service, and any client that joins rooms.

This guide doesn't cover Redis, TURN, the SIP service, or agents. Redis and the SIP service are in `livekit-sip-service.md`. TURN is only needed if some clients can't connect.

---

## 2. Requirements

| Requirement | Notes |
|---|---|
| VPS with a public IP | Any Linux server with Docker. Check CPU and RAM against how many calls you expect. |
| Coolify installed | Running and reachable from your browser |
| A domain or subdomain | Pointing at the VPS's public IP |
| Swap space | Strongly recommended on small servers (section 3) |

**Sizing:** LiveKit's guidance is roughly 100–150 two-way participants per 4-core server for small rooms. Voice-only calls are lighter. Plan for fewer on a 2-core server, and leave headroom for your other apps.

---

## 3. Add swap first

Small servers can run out of memory during deploys, and the kernel then kills processes. Adding swap is a safety net.

Run as root on the VPS:

```bash
fallocate -l 4G /swapfile
chmod 600 /swapfile
mkswap /swapfile
swapon /swapfile
echo '/swapfile none swap sw 0 0' >> /etc/fstab
echo 'vm.swappiness=10' > /etc/sysctl.d/99-swappiness.conf
sysctl -p /etc/sysctl.d/99-swappiness.conf
```

Verify:

```bash
swapon --show
free -h
```

Note: some systems don't have `/etc/sysctl.conf`, which is why the swappiness setting goes in a drop-in file under `/etc/sysctl.d/`.

---

## 4. Generate the key pair

The key pair is created by you. It isn't provided by LiveKit Cloud.

On Windows, in PowerShell:

```powershell
$chars = (48..57) + (65..90) + (97..122) | ForEach-Object { [char]$_ }
$key = "API" + -join (1..12 | ForEach-Object { $chars | Get-Random })
$secret = -join (1..40 | ForEach-Object { $chars | Get-Random })
"LIVEKIT_KEYS=${key}: $secret"
```

On Linux or macOS:

```bash
echo "API$(openssl rand -hex 6)"
openssl rand -hex 24
```

The key is an identifier. The secret signs access tokens, so anyone with it can create room access on your server. Use the same pair everywhere: the LiveKit server, the token script, and any agent or SIP service.

---

## 5. Create the LiveKit resource in Coolify

1. In your project and environment, add a **Docker Image** resource.
2. **Image:** `livekit/livekit-server`
3. **Tag:** `latest` for now. Pin a specific version once it works.
4. **SHA256 / digest:** leave empty.
5. **Domain:** `<your-domain>`, with HTTPS enabled. Coolify issues the certificate.

### Ports

| Setting | Value | Why |
|---|---|---|
| Ports exposes | `7880` | LiveKit's HTTP and WebSocket port, which Coolify's proxy forwards to your domain |
| Port mappings | `7881:7881,50000-50100:50000-50100/udp` | TCP fallback for media and a UDP range for WebRTC audio and video |

**Port mappings vs. Ports exposes:** Ports exposes is for the domain proxy (web traffic). Port mappings publish ports directly on the VPS (media traffic). They do different jobs.

Keep the UDP range small. Each published port adds a Docker proxy process, so a very large range uses a lot of memory. Size the range to your expected number of simultaneous connections.

### Environment variables

| Variable | Value |
|---|---|
| `LIVEKIT_KEYS` | `<api-key>: <api-secret>` (the key and secret separated by a colon and a space) |
| `LIVEKIT_CONFIG` | See below |

Set `LIVEKIT_CONFIG` as one line, matching the port range you published:

```
{rtc: {tcp_port: 7881, port_range_start: 50000, port_range_end: 50100, use_external_ip: true}}
```

What the RTC settings do:
- `tcp_port`: the TCP media fallback port
- `port_range_start` / `port_range_end`: the UDP range. It must match the port mapping.
- `use_external_ip: true`: makes LiveKit advertise the VPS's public IP. Without it, LiveKit advertises its private Docker address, and clients outside the VPS can't connect to media.

**Formatting rules:**
- Keep a space after every colon.
- Don't include the variable name in the value if the field already has it.
- Avoid line breaks inside the value.

### Network

Keep Coolify's default network. Don't use host networking, since the port mappings publish the media ports.

### Redis and command

Redis is optional for a single test server. Add it later when you need the SIP service or multiple nodes. See `livekit-sip-service.md` for the Redis settings.

Leave the command field empty. The image starts LiveKit with the settings above.

---

## 6. Open the firewall

Allow these on the VPS, and on your provider's firewall panel if it has one:

| Port | Protocol | Purpose |
|---|---|---|
| 80 and 443 | TCP | HTTP and HTTPS, for Coolify and Let's Encrypt |
| 7881 | TCP | LiveKit media fallback |
| 50000–50100 | UDP | WebRTC media |

Check the current state before changing anything:

```bash
ufw status verbose        # if ufw is installed
iptables -S INPUT         # shows the default policy
```

---

## 7. Deploy and verify

Start the deploy in Coolify and watch its logs. A healthy start looks like this:

```
using single-node routing
found external IP via STUN ... "externalIP": "<vps-ip>"
using external IPs ... "advertiseInternalIP": false
starting LiveKit server ... "rtc.portTCP": 7881, "rtc.portICERange": [50000, 50100]
```

If you see `one of key-file or keys must be provided`, the `LIVEKIT_KEYS` variable is missing or malformed.

On the VPS:

```bash
# Container status
docker ps --format '{{.Names}}\t{{.Status}}\t{{.Image}}' | grep livekit

# Published ports
docker port <container-name>

# Ports listening
ss -lntu | grep -E ':7880|:7881'

# Memory
free -h
dmesg -T | grep -i 'out of memory'
```

From your own computer:

```powershell
Test-NetConnection <vps-ip> -Port 7881
```

Then check the domain:

```bash
curl -s -o /dev/null -w "%{http_code}\n" https://<your-domain>/
```

It should return HTTP 200.

---

## 8. Problems and fixes

| Symptom | Cause | Fix |
|---|---|---|
| Container keeps restarting with `one of key-file or keys must be provided` | `LIVEKIT_KEYS` missing or malformed | Set `LIVEKIT_KEYS` in the format `<key>: <secret>` |
| Domain returns 502 Bad Gateway | Coolify's proxy is forwarding to the wrong port | Set Ports exposes to `7880` |
| Domain returns 503 | No running container for the proxy to reach | Check that the container is running, and check its logs |
| Clients connect for signaling but not media | Media ports not published, or the private IP is advertised | Set port mappings, and set `use_external_ip: true` |
| Startup log shows a different port range than configured | The `LIVEKIT_CONFIG` value wasn't saved or was malformed | Check the value and redeploy |
| Container stuck in `Created` | The deploy didn't finish | Check the deploy log in Coolify, cancel, and redeploy |
| Memory spikes during a deploy, other apps get killed | Little or no swap, plus the deploy restarting other containers | Add swap (section 3) and deploy when memory is stable |
| Many `docker-proxy` processes | A large published port range | Reduce the UDP range |

---

## 9. Lessons

- **Check memory before and during a deploy.** `free -h` and `dmesg -T | grep -i 'out of memory'` show whether the server is struggling.
- **Check the running container, not only the Coolify settings.** `docker inspect <container> --format '{{json .HostConfig.PortBindings}}'` shows what was actually applied.
- **Verify environment variable names against the docs.** Use the documented names, and confirm them in the logs after startup.
- **Quotes and line breaks can corrupt values.** Coolify may save quotes with literal backslashes. Keep values on one line, and check the saved value.
- **Keep the UDP range small** for a prototype, and widen it only when you need more simultaneous connections.

---

## 10. Next steps

- Point your agents at `wss://<your-domain>` with the same key pair.
- Add Redis and the SIP service when you need phone calls. See `livekit-sip-service.md`.
- Pin the LiveKit image to a specific version once the setup is stable.
- Add TURN if some clients can't connect through the standard paths.
