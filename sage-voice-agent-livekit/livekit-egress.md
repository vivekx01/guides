# LiveKit Egress: what it does and how we could use it

Egress is LiveKit's recording and export service. It takes audio and video from a room or from individual tracks and saves it as a file, or streams it to another destination. It's a separate service from the LiveKit server, and it's open source, so it can run on your own VPS.

This document covers what Egress can do, what it needs on our server, and how we might use it with Sage. It also marks which points we've checked and which still need testing.

Placeholders in `<angle brackets>` stand for your own values.

---

## 1. Where Egress fits

```
LiveKit server (VPS)  ──room and track events──▶  Egress service (VPS)
                                                      │
                                                      ▼
                                       files (MP4, OGG, etc.) or streams
                                       saved to storage you configure
```

Egress doesn't change how calls work. It listens to the room, copies the media, and writes it somewhere. People and Sage don't need to do anything differently.

---

## 2. Egress types

LiveKit's documentation describes these types. Confirm the details against the current docs before building on them.

| Type | What it records | Notes |
|---|---|---|
| **Room composite** | Everyone in a room, mixed into one video or audio file | Starts a headless Chrome browser to render the room, so it's heavy on CPU |
| **Web** | A web page you point it at | Records any page, not only LiveKit rooms |
| **Track composite** | One participant's audio and video tracks, synchronized | Lighter than room composite |
| **Track** | A single track, saved on its own | Lightest option for audio-only capture |
| **Participant** | One participant's audio and video together | Newer and simpler than track composite |
| **Auto egress** | Starts recording automatically when a room is created | Configured in the room settings |

### Output formats

Room composite, web, and track composite can produce:
- **MP4 and OGG files**
- **HLS segments** (for live playback)
- **RTMP or SRT streams** (for live streaming platforms)
- **JPEG thumbnails**

Track egress can produce:
- **MP4, OGG, and WebM files**
- **WebSocket streams**

For Sage, the useful options are **audio-only** outputs, such as an OGG or MP4 file containing only the caller's audio.

---

## 3. What Egress needs on our server

| Requirement | Why | Status for us |
|---|---|---|
| **LiveKit server v1.13.5 or later** | Required for self-hosted Egress | We run 1.13.7, so this is met |
| **A separate Egress container** | Egress runs as its own service | Not set up yet |
| **Redis** | Egress and the server share state through Redis | Not set up yet. Our server currently runs in single-node mode without it |
| **The same API key pair** | Egress authenticates with your LiveKit keys | Already in place |
| **Storage** | Somewhere to save the output files | Not chosen yet. Options: a local folder on the VPS, or an S3-compatible bucket |
| **Enough memory and CPU** | Room composite runs Chrome, which is heavy | Check `free -h` and CPU load on your server before enabling it, and leave headroom for your other apps |

---

## 4. Ways we could use it with Sage

### Option A: record the caller's audio only

Save each caller's audio as its own file, separate from Sage's replies. This is the lightest option and the most useful for reviewing calls.

- **Egress type:** track or participant egress, audio only
- **Output:** OGG or MP4 file per call
- **Cost on the server:** low

### Option B: record the whole conversation

Mix the caller and Sage into one audio file. This gives a complete record of the call.

- **Egress type:** room composite, audio only
- **Output:** one audio file per room
- **Cost on the server:** moderate. Confirm that audio-only room composite avoids the Chrome pipeline before relying on it.

### Option C: automatic recording

Turn on auto egress so every room is recorded from the moment it's created. This suits production but needs storage and clear consent rules first.

### Option D: transcripts without audio

Sage can save each turn's text directly. This needs no Egress service and very little storage. It's the simplest option, but it doesn't keep the audio.

---

## 5. Consent and legal points

- **Tell callers they're being recorded** before the recording starts, and give them a way to decline.
- **Recording rules vary by location.** Some places require every party to consent. Check the rules for where you and your callers are.
- **Keep recordings secure.** Limit who can access the storage, and delete recordings you no longer need.

---

## 6. What we'd need to set up

1. **Choose storage.** A local folder on the VPS is the simplest for a prototype.
2. **Add Redis** to Coolify, in the same environment as LiveKit.
3. **Add the Egress container** to Coolify, using the same key pair and the same Redis.
4. **Choose the recording type** (Option A is the recommended starting point).
5. **Start a test recording** with a short call, then check the output file and the VPS memory during the recording.
6. **Add consent messaging** to Sage before recording real callers.

---

## 7. What we haven't verified yet

- Exact Coolify steps for the Egress container.
- Whether audio-only room composite avoids Chrome. This needs testing.
- How much memory and CPU Egress uses on your VPS.
- How to start a recording from our code, including the exact API calls.
- The storage options available on your VPS.

Each of these should be checked against LiveKit's current documentation before we rely on it.

---

## 8. Glossary

- **Egress:** LiveKit's recording and export service.
- **Composite:** a single file made by combining several tracks or participants.
- **Track:** one stream of audio or video from one participant.
- **Headless Chrome:** a browser without a window, used to draw the room for recording.
- **HLS:** a streaming format that splits video into small segments.
- **RTMP and SRT:** protocols for sending live video to streaming platforms.
- **Auto egress:** recording that starts automatically when a room is created.
