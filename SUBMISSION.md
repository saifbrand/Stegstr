# Stegstr — contest submission

The headline: **the hidden data now survives being resized, which it previously
never did.** On modelled WhatsApp, Instagram, Telegram, Facebook and Twitter
channels, the shipping encoder recovers **100% of payloads (84/84)**, against
**71.4%** before — and the two platforms that previously scored a flat **0%**
now score 100%.

Everything below is reproducible from this repo. No Rust toolchain is needed.

**Try it in a browser:** <https://stegstr-app.pages.dev> — a hosted build of this
app, for a first look without cloning. Drop in a photo, embed a message,
download it, send it through WhatsApp, drop the returned file back in. Nothing
about the steganography depends on the server, so the same code works offline
and locally. Judge the pinned commit rather than the demo: the hosted copy may
lag it, and it is the local build that this write-up measures.

---

## 1. Run it

**Fastest check — nothing to install.** The bundled CLI is self-contained, so
straight after unzipping, with only Node 18+ on the machine:

```bash
node dist-cli/stegstr.mjs selftest
node dist-cli/stegstr.mjs detect testkit/out/landscape-locator.jpg
node dist-cli/stegstr.mjs embed <your-photo.jpg> -o secret.jpg --payload "meet at six"
node dist-cli/stegstr.mjs detect secret.jpg
```

**The app:**

```bash
npm install
npm run dev            # app on http://localhost:5173
```

Confirm the whole stack in about a minute:

```bash
npm test               # 87 unit tests
npm run build:cli      # rebuild the CLI from source
npm run test:browser   # builds the app, then drives a real Chromium round trip
```

## 2. Check the robustness claim yourself

Do not take the numbers on trust — the kit exists so you can settle it with
your own phone:

```bash
npm run testkit                      # writes 8 images to testkit/out/
# Send them to yourself via WhatsApp / Telegram / Instagram as normal photos.
# Save what comes back into returned/
npm run testkit:verify returned/
```

The verifier matches on recovered payload rather than filename, because the
platforms rename what they hand back.

**One trap worth knowing before you run this**, because it cost me a test run:
if you send the images to yourself on WhatsApp Web and download them in that
same session, the files come back **byte-identical** — WhatsApp hands you your
own original rather than the server's recompressed copy, and everything passes
for the wrong reason. Upload and download have to happen on **different
devices**: send from the phone, download on the desktop, or have someone else
send them back. The giveaway is file size — a genuine round trip through
WhatsApp shrinks a 586 kB image to roughly a sixth of that. If the sizes come
back unchanged, the test did not actually happen.

To reproduce the measured matrices instead:

```bash
cd robust_lab && pip install -r requirements.txt
python bench.py current              # the old encoder, as a baseline
python crossvalidate.py --mode locator   # the shipping encoder, end to end
```

## 3. Against the contest's own gauntlet

`robust_lab/gauntlet.py` replicates the published methodology (Evaluator
Instructions v1.0, section 2): a known 43-byte secret, a control check, five
lossy profiles from light to heavy plus a resize profile, and PSNR/SSIM of stego
against cover. Run it with `python robust_lab/gauntlet.py`.

| cover | control | Light | Mod. | Strong | Heavy | Resize | PSNR | SSIM |
|-------|---------|-------|------|--------|-------|--------|------|------|
| landscape | PASS | OK | OK | OK | OK | OK | 40.7 | 0.967 |
| foliage   | PASS | OK | OK | OK | OK | OK | 39.8 | 0.965 |
| portrait  | PASS | OK | OK | OK | OK | OK | 41.0 | 0.967 |
| night     | PASS | OK | OK | OK | OK | OK | 39.8 | 0.968 |

**Survival 20/20 (100%). PSNR 40.3 dB** with the default JPEG output, **42.6 dB**
writing PNG (`-o out.png`), SSIM 0.976.

### One number I deliberately did not chase

Dropping the dither step to 24 raises PSNR to 49.3 dB while still scoring 100%
on these five profiles, which would look better on a board. I did not ship that,
because it fails a case the profiles do not cover: **WhatsApp Web and Telegram
Desktop resize in the client**, with a filter that aliases rather than averages,
and only the shipping step of 56 survives it. Measured in Chromium, a 1600 to
1080 canvas downscale decodes at step 56 and fails at 48 and below. Section 5E
calls real transference the heart of the contest, so tuning the visible metric
at the cost of the real path was the wrong trade.

What I did take was free: the distortion measured against the cover is dominated
by the *output's own* JPEG compression, not by the watermark. Raising output
quality from 92 to 98 gains 1.8 dB and writing PNG gains 4.1 dB, both with
survival unchanged.

## 4. What was wrong, and what changed

### The root cause nobody had isolated

`docs/WHATSAPP_PLAN.md` recorded the open problem as *"QIM passes our simulated
WhatsApp but fails on real WhatsApp"* and concluded the simulator's quality
factor must be wrong. It was the wrong suspect. Isolating the two things a
platform does to an image:

| Cover size | Platform  | Did it resize?  | Decode |
|------------|-----------|-----------------|--------|
| 1600×1200  | whatsapp  | no              | PASS   |
| 1600×1200  | telegram  | yes → 1280×960  | FAIL   |
| 2000×1500  | whatsapp  | yes → 1600×1200 | FAIL   |
| 1608×1208  | whatsapp  | yes → 1600×1202 | FAIL   |

**Recompression was always survivable. Any resize at all was fatal** — the last
row is a 0.5% resize that still destroyed 100% of payloads.

The reason is structural: the old encoder embedded in the 8×8 DCT blocks of the
image *at its delivered resolution*, so resampling misaligned the receiver's
grid and blended every coefficient. That also explains why `dct_rs64` showed no
gain over `dct` in the old comparison — parity was never the limiting factor.
The existing mitigation (pre-resize the cover to a chosen platform's exact cap)
only worked when the user guessed the destination correctly.

### The fix

Both ends normalise the luma plane onto a fixed canonical square before doing
anything, so a rescale is undone by the receiver's own normalisation. Inside
that frame, each payload bit rides on the *projection* of a pseudorandom set of
mid-frequency coefficients onto a ±1 chip vector (spread-transform dither
modulation), which cancels host interference and spreads each bit across many
coefficients. The usable frequency band was chosen by measuring each channel's
response (`robust_lab/probe_band.py`), not by intuition.

| Scheme        | payload | tg-file | instagram | facebook | twitter | telegram | whatsapp | wa ×2 | **Overall** |
|---------------|---------|---------|-----------|----------|---------|----------|----------|-------|-------------|
| old (QIM)     | 64 B    | 100%    | **0%**    | 100%     | 100%    | **0%**   | 100%     | 100%  | **71.4%**   |
| stdm:locator  | 48 B    | 100%    | 100%      | 100%     | 100%    | 100%     | 100%     | 100%  | **100%**    |
| stdm:standard | 163 B   | 100%    | 100%      | 100%     | 100%    | 100%     | 100%     | 100%  | **100%**    |

Also **13× faster to embed** (0.20 s vs 2.71 s per image).

Arbitrary rescaling — the thing that broke everything — is now a non-event:

| Attack | new | old |
|--------|-----|-----|
| scale 37% / 63% / 150% | **PASS** | FAIL |
| resize to non-multiple-of-8 | **PASS** | FAIL |
| JPEG Q30 | PASS | PASS |
| greyscale | PASS | PASS |
| crop 2% per edge | **PASS** | FAIL |

### A failure only running the app could find

Every channel model resizes with a good filter, because that is what a
platform's servers do. Driving the built app in Chromium exposed a case none of
them covered: **WhatsApp Web and Telegram Desktop resize in the client**, with a
canvas bilinear filter that aliases instead of averaging. At the original dither
step the payload did not survive that. Raising the step fixed it for under half
a decibel of PSNR, because at this scale JPEG's own quantisation noise dominates
the watermark's. `npm run test:browser` now covers this path.

### Networking

The relay layer had three quiet failure modes, all now fixed and covered by
tests in `src/__tests__/relay.test.ts`:

1. **The relay list was fetched from `stegstr.com` with no timeout.** That host
   is unreachable as of this writing, so every launch blocked on two TCP
   timeouts before falling back to defaults. Both attempts are now bounded.
2. **Dropped sockets were never re-established.** `onclose` cleared the handle
   and nothing reconnected, so a feed that worked at launch silently stopped
   updating. Connections now retry with exponential backoff and jitter, and
   re-send their subscriptions on reopen.
3. **"Synced" required *every* relay to answer**, so one unreachable relay left
   the UI on "Connecting…" forever — while one *failing* relay was enough to
   paint the whole status as an error. Both are now decided across the pool,
   with a timeout backstop.

Also: events arriving from several relays are de-duplicated before reaching the
UI, and `publishEvent` now reports each relay's actual verdict — a post that no
relay accepted used to be indistinguishable from one that all of them did.

### AI agent operability

- **Node CLI** (`dist-cli/stegstr.mjs`) — `embed`, `detect`, `capacity`,
  `modes`, `selftest`. Every command speaks `--json`; exit codes are meaningful.
  No compiler in the loop, so an agent can go from clone to working in one step.
- **`window.stegstr`** — the running app exposes the same operations, so a
  browser driver can use the GUI build headlessly. See `src/agent-api.ts`.
- **Updated skill manifest** (`skill/stegstr/SKILL.md`) describing the modes,
  the JSON contract, and — importantly — what does *not* survive.

### A latent private-key leak

`nostr-stub.ts` exported two event builders. The async one signs properly. The
synchronous one faked both fields:

```js
ev.id  = Math.abs(h).toString(16)...          // a 32-bit string hash, not SHA-256
ev.sig = bytesToHex(secretKey).slice(0, 128)  // the secret key, as the signature
```

Nothing called it, so no key was ever published, but it was exported and shaped
exactly like the real thing: one future caller away from broadcasting a user's
private key to every relay it connects to. It now signs for real, synchronously,
since `sha256` from `@noble/hashes` never needed to be async in the first place.
Six tests in `src/__tests__/nostr-signing.test.ts` check what a relay checks -
that the id is the NIP-01 hash of the event's own content, that the signature
verifies against its pubkey, and that a tampered event fails.

### One finding about the project's own infrastructure

`https://www.stegstr.com/config/relay.json` serves no `Access-Control-Allow-Origin`
header, so a browser build can never read it; the request is blocked before the
response is seen. The app falls back to its built-in relay list, so nothing
breaks, but the hosted configuration is currently unreachable from the web.

### Other fixes

- `embed` now verifies its own output before reporting success. Silently
  handing someone an image that carries nothing was the worst available
  outcome.
- Images too small for the selected mode are refused with the required size,
  rather than producing an unreadable file.
- `stego-crypto.ts` logged the IV, leading ciphertext bytes and plaintext length
  to the console on every call. Removed — needless disclosure in a tool whose
  purpose is to avoid being noticed.
- Images made by older builds still open: the legacy QIM and Dot readers remain
  in the detect path.

## 5. Modes

Capacity and robustness genuinely trade against each other, so the app exposes
the choice instead of hiding it:

| Mode | Payload | Min edge | For |
|------|---------|----------|-----|
| `locator` | 48 B | 320 px | WhatsApp/Instagram. Carries a 32-byte Nostr event id + 16-byte key; the post syncs over relays. |
| `standard` | 163 B | 384 px | A short note carried entirely inside the image. |
| `bulk` | 1377 B | 960 px | Lossless channels only: Telegram "send as file", email, disk. |

`locator` is what ties the two contest requirements together: the
steganographic channel carries a *pointer* and the networking layer resolves
it, which is what makes 100% WhatsApp survival possible without pretending a
64 kB bundle can survive a 1080 px downscale.

## 6. Known limits

Stated plainly, because a scheme whose limits are undocumented is worse than one
whose limits are known:

- **Rotation is not handled**, not even 0.5°.
- **Crops beyond ~2% per edge are not handled.** The detector re-expands by a
  few candidate insets, which works while the missing border is thin. Both this
  and rotation need a synchronisation template (log-polar / Fourier–Mellin)
  rather than a parameter change.
- **PSNR is 34–41 dB**, not the >40 dB of a fragile scheme. That is the honest
  price of surviving a Q65 recompression, and it is in the normal range for
  robust watermarking.
- **The channel models are models, and the headline number comes from them.**
  They are materially harsher than the previous ones — long-edge caps,
  non-multiple-of-8 output, sampled quality ranges, sharpening, double-encode,
  edge crop — and the browser finding above closed the one real-world gap I
  could find. But they are not the real apps, and I am not going to present a
  simulated result as a measured one. The test kit in section 2 exists so this
  can be settled with your own account rather than argued about.
- **Benchmark covers are synthetic**, built with 1/f statistics to behave like
  photographs across smooth, detailed and dark cases. A real photo set should be
  run before quoting these figures anywhere else.

## 7. Where things live

| Path | What |
|------|------|
| `src/stego-stdm.ts` | The encoder. Canvas-free, so it runs in Node and is unit-testable. |
| `src/dct2d.ts` | FFT-backed 2-D DCT for the canonical grid. |
| `src/stego-stdm-web.ts` | Browser bindings (canvas in, JPEG out). |
| `src/agent-api.ts` | `window.stegstr`. |
| `src/relay.ts` | Nostr relay client, rewritten for reliability. |
| `cli/` | Node CLI. |
| `testkit/` | Real-device test kit and verifier. |
| `robust_lab/` | Channel models, benchmarks, band probe, and `RESULTS.md`. |
| `src/__tests__/` | 87 unit tests, including the relay failure modes and Nostr signing. |
