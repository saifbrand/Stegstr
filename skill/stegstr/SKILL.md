---
name: stegstr
summary: Hide and recover messages in images so they survive WhatsApp, Telegram and Instagram. Steganographic Nostr client, works offline, no registration.
description: Embed and detect Stegstr payloads in images. Use when the user needs to hide data in a photo, extract hidden data from one, or work with steganographic social networking (Nostr-in-images). Payloads survive platform recompression and resizing. Node CLI with JSON output for scripts and agents; no compiler required.
license: MIT
tags: steganography, nostr, images, crypto, integration, file-management, automation, cli
install:
  requirements: |
    - Node.js 18 or newer
    - Git
  steps: |
    1. git clone https://github.com/brunkstr/Stegstr.git
    2. cd Stegstr
    3. Run it: node dist-cli/stegstr.mjs selftest
       (the bundled CLI is self-contained - no npm install needed)
    4. To rebuild from source: npm install && npm run build:cli
permissions:
  - filesystem
metadata:
  homepage: https://stegstr.com
  repo: https://github.com/brunkstr/Stegstr
---

# Stegstr

Stegstr hides messages and arbitrary payloads inside images. Unlike naive
steganography, the payload is designed to survive being sent through a real
messaging platform: the encoder normalises the image before embedding, so
resizing and re-compression by WhatsApp, Telegram or Instagram do not destroy it.

## When to use this skill

- Hide a message, JSON, or arbitrary bytes inside an image.
- Extract hidden data from an image that may have been through a chat app.
- The user mentions steganography, Nostr-in-images, Stegstr, hiding data in
  photos, or secret messages in pictures.
- Programmatic or agent-driven use.

## CLI

```bash
node dist-cli/stegstr.mjs selftest      # confirm it works; no install required
```

Every command accepts `--json`. Exit codes: `0` success, `1` no payload found,
`2` usage or input error. Prefer `--json`: it is stable, and the human output is
not.

### Embed

```bash
node dist-cli/stegstr.mjs embed cover.jpg -o out.jpg --payload "secret text"
node dist-cli/stegstr.mjs embed cover.jpg -o out.jpg --payload-file note.json --mode standard
node dist-cli/stegstr.mjs embed cover.jpg -o out.jpg --payload-base64 <b64> --mode locator
```

The command decodes its own output before reporting success, so `ok: true`
means the payload was verified present, not merely written.

### Detect

```bash
node dist-cli/stegstr.mjs detect image.jpg --json
```

Tries every mode unless `--mode` names one. Returns the payload as `utf8` when
it is text and `base64` when it is not; check the `encoding` field rather than
guessing.

### Capacity and modes

```bash
node dist-cli/stegstr.mjs modes --json
node dist-cli/stegstr.mjs capacity cover.jpg --mode standard --json
```

## Modes

Capacity and robustness trade against each other, so pick deliberately:

| Mode       | Payload  | Min image edge | Use for |
|------------|----------|----------------|---------|
| `locator`  | 48 bytes | 320 px         | Anything going through WhatsApp or Instagram. Sized to carry a reference (a 32-byte Nostr event id plus a 16-byte key); the message itself syncs over relays. |
| `standard` | 163 bytes| 384 px         | A short note carried entirely inside the image. Survives the mainstream platforms. |
| `bulk`     | 1377 bytes | 960 px       | Lossless delivery only: Telegram "send as file", email, disk. Will not survive recompression. |

`locator` is the default and the one to use when robustness matters.

## Driving the running app

The app exposes the same operations on `window.stegstr`, so a browser
automation driver can use the GUI build headlessly:

```js
await window.stegstr.encode(coverBlob, "secret", "standard"); // -> Blob (JPEG)
await window.stegstr.decode(imageBlob);                       // -> string | null
await window.stegstr.decodeDetailed(imageBlob);               // -> { ok, payload, mode }
window.stegstr.modes();                                       // -> capacity per mode
```

## What survives, and what does not

Measured against modelled platform channels and confirmed on the shipping
encoder (see `robust_lab/RESULTS.md`):

- **Survives:** re-compression down to JPEG Q60, resizing by any factor,
  metadata stripping, chroma subsampling, greyscale conversion, being forwarded
  through two apps in a row, and small aspect-ratio crops.
- **Does not survive:** rotation, crops larger than about 2% per edge, heavy
  filtering, or screenshots at reduced resolution.

If a decode fails, the usual cause is one of those, not a bad image.

## Image formats

Input may be PNG or JPEG. Output should be `.jpg`; PNG output is written
losslessly but platforms convert it to JPEG anyway. Images smaller than a mode's
minimum edge are refused rather than silently producing an unreadable file.

## Payload format

`SGX1` magic, a compression flag, a two-byte length, then the payload,
Reed-Solomon encoded into a fixed-size frame. Payloads are deflate-compressed
when that helps. The app additionally encrypts before embedding; the CLI embeds
what it is given, so encrypt first if you need confidentiality.
