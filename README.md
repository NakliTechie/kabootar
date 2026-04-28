# 🕊️ Kabootar — कबूतर

A browser-native LocalSend in a single HTML file. Two devices on the same WiFi open the page, scan each other's QR — file transfers peer-to-peer over WebRTC. No app to install. No accounts. No server. No relay.

**[→ Try it live](https://kabootar.naklitechie.com)**

## What it does

Open Kabootar on a sender device. Pick a file. The page generates a WebRTC offer, compresses it, renders it as a QR. Open Kabootar on a receiver device. Point its camera at the sender's QR. The receiver generates an answer, renders it as another QR. Sender's camera reads the answer back. ICE completes via host candidates / mDNS — both browsers resolve each other's `xxxxx.local` candidates locally — and the file streams over the WebRTC data channel in 14 KB chunks. DTLS encrypts the wire end-to-end.

Once the page is loaded, **zero outbound network requests** are made. The pairing is a closed loop between the two cameras and screens in front of you.

## How to run

Serve any static way:

```bash
python3 -m http.server 8121
# open http://localhost:8121/
```

Or open the deployed site at https://kabootar.naklitechie.com.

**HTTPS is required** for the camera (`getUserMedia`) and the streaming-save (`showSaveFilePicker`). `http://localhost` works for desktop development; LAN IPs over plain HTTP do not.

No build step. Single HTML file. Two CDN deps lazy-loaded once and cached: [qrcode@1.5.4](https://www.npmjs.com/package/qrcode) for encoding, [jsqr@1.4.0](https://www.npmjs.com/package/jsqr) as the iOS Safari decoder fallback (Chromium uses the native `BarcodeDetector`).

---

## Why would you need this?

LocalSend is wonderful — it's the polished cross-platform binary that does this. The trade-off is that it's a binary, which means an install per device, app-store policies, version drift, and a folder of native code per OS.

Kabootar is the same idea constrained to what a browser tab can do without any installation:

- **Nothing to install.** Open a URL on each device. That's the entire setup.
- **Nothing leaves the LAN.** Once the page is loaded, the WebRTC connection finds host / `xxxxx.local` candidates and stays on your local network. No STUN, no TURN, no relay, no telemetry.
- **End-to-end encrypted.** WebRTC's DTLS handshake encrypts the data channel. The QR codes carry the SDPs that bootstrap that handshake.
- **Multi-GB on capable browsers.** Chromium desktop streams to disk via `FileSystemWritableFileStream` — no in-memory ceiling. Safari, Firefox, iOS, and Android fall back to in-memory `Blob` with conservative caps (500 MB / 200 MB on iOS).
- **Resume in-session** *(Slice 2 — coming).* Drop your WiFi mid-transfer and re-pair via QR — receiver tells sender which chunks it has, sender re-sends only the gaps.

The trade-offs are honest:

- **No auto-discovery.** Browsers can't do mDNS or UDP multicast. Two devices need a manual pairing gesture; ours is a QR scan. Both devices are physically together (same WiFi, same room) so a QR scan is a natural cue.
- **Both devices need a camera** for the primary flow. Copy/paste mode is provided as a fallback for desktop devices without a webcam.
- **Symmetric NAT pairs across different networks won't connect** without a TURN server. We don't ship one. The intended target is "same WiFi" — that always works.

---

## How it works

```
┌────────────────────┐         ┌─────────────────────┐
│  Sender device     │         │  Receiver device    │
│                    │         │                     │
│  Pick file         │         │                     │
│  Generate offer    │         │                     │
│  Compress + QR ⌐── │ ─── 📷 ─→ Scan, decode        │
│                    │         │  Generate answer    │
│  Scan, decode  ←─ 📷 ─── │ ──┘ Compress + QR        │
│                    │         │                     │
│  WebRTC connect over LAN (host / xxxxx.local)      │
│                    │         │                     │
│  Frame file → 14 KB chunks → data channel          │
│                    │         │                     │
│                    │         │  Stream to disk     │
│                    │         │  or in-memory blob  │
└────────────────────┘         └─────────────────────┘
```

The wire framing per packet:

| Tag    | Purpose          | Payload                                |
|--------|------------------|----------------------------------------|
| `0x01` | Manifest         | JSON `{name, size, mime, total, sha256}` |
| `0x02` | Chunk            | u32-LE seq · raw bytes (≤ 14 KB)       |
| `0x05` | Done             | (empty)                                |

Chunk size is 14 KB because the WebRTC ~16 KB safe-message ceiling applies to the **framed** packet (overhead included), not the body — getting this wrong silently drops messages with no error trail. ([CONV.md](https://github.com/NakliTechie/babellocal) for the gory details across the series.)

## Verify

The repo's debug surface (`window.__kabootar`, always on) exposes the protocol primitives so you can drive the full flow from DevTools without a real camera:

```js
const K = window.__kabootar;
// 1. Build a real offer SDP
const inviter = await K.WebRTCTransport.asInviter();
const compressed = await K.SDPCodec.compress(inviter.localSdp());
// 2. Round-trip it through the codec
const back = await K.SDPCodec.decompress(compressed);
console.assert(back === inviter.localSdp());
// 3. Build the answer side
const joiner = await K.WebRTCTransport.asJoiner(back);
await inviter.acceptAnswer(joiner.localSdp());
await Promise.all([inviter.opened, joiner.opened]);
// connection is live — sendFile / receiveOnTransport take it from here
```

There's also an in-tab loopback page (`#/loopback`) that exercises the framing + chunking + backpressure + Tier-A streaming save without any pairing — useful for verifying the Slice-0 protocol layer on each new browser before testing the cross-device flow.

---

## Tech

- **WebRTC data channel** with `iceServers: []` — pure-LAN, no STUN, no TURN, no relay. Chromium's automatic mDNS resolution handles the `xxxxx.local` candidates.
- **`CompressionStream("deflate-raw")`** + base64url for the SDPs — no extra deps for compression; native Web API.
- **`BarcodeDetector`** for QR scanning (Chrome, Edge, Android Chrome, Safari 16.4+) → **jsQR** lazy fallback (~12 KB) for iOS Safari.
- **`FileSystemWritableFileStream`** for streaming saves on Chromium desktop — multi-GB fine, never holds the file in memory.
- **`<input type="file">` + `URL.createObjectURL` + `<a download>`** for the universal sender + Tier-B receiver path.

Zero build tooling. One HTML file.

## Palette

Coloured with **`india1-06 · नील NEEL`** — Bagru block-print indigo, the dye Gandhi went to satyagraha for in 1917 in Champaran. Trustworthy blue body, sky-blue panels, deep ink, brick-red brand mark, indigo action button. Fitting for a tool whose whole identity is "messenger across distance."

Palette pulled from [**Rangrez**](https://github.com/NakliTechie/Rangrez), the colour-palette library that backs all NakliTechie projects.

---

## Part of the NakliTechie series

| Tool | What it does |
|------|--------------|
| **Kabootar** | Browser-native LocalSend — QR-paired P2P file send |
| [**Mehfil**](https://github.com/NakliTechie/mehfil) | Browser-native team chat — E2E encrypted, no central server |
| [**BabelLocal**](https://github.com/NakliTechie/BabelLocal) | Offline translation — 200 languages, NLLB model |
| [**Tijori**](https://github.com/NakliTechie/Tijori) | Browser-native password vault — multi-device sync |
| [**KanZen**](https://github.com/NakliTechie/KanZen) | Kanban without the noise — single-file local-first |
| [**NakliPoster**](https://github.com/NakliTechie/NakliPoster) | Local-first API client — Postman alternative |
| [**BOFH**](https://github.com/NakliTechie/BOFH) | Browser-native dev toolkit — 20 modules |

Full list at [naklitechie.github.io](https://naklitechie.github.io/).

---

**Built by [Chirag Patnaik](https://github.com/NakliTechie)**

---

*Built in a few hours with [Claude](https://claude.ai).*
