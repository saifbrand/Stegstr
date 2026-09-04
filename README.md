# Stegstr

**Steganographic social networking.** Hide messages in images and share them anywhere—local-first, with optional Nostr sync.

Hidden data **survives being sent through WhatsApp, Telegram and Instagram**: the
encoder normalises the image before embedding, so the resizing and
re-compression those platforms apply do not destroy the payload. See
[SUBMISSION.md](SUBMISSION.md) for what changed and the measurements, and
[robust_lab/RESULTS.md](robust_lab/RESULTS.md) for how to reproduce them.

Stegstr gives you two ways to use it:

- **UI app** — Desktop and mobile app. Create posts, embed them in images, and detect content from images with a graphical interface.
- **CLI module** — Command-line tool for scripts and automation. Embed, detect, and inspect capacity from the terminal, with JSON output for agents.

Both use the same steganographic format. Data is stored and processed **locally**; Stegstr is **not exclusively Nostr**. You can use it fully offline (embed/detect in images and share via any channel). When you want to sync over the network, Stegstr can act as a Nostr client and use relays.

## Quick start

### Graphical app (UI)

Download the latest release for your platform:

- [macOS](https://github.com/brunkstr/Stegstr/releases/latest/download/Stegstr-macOS.dmg) · [Windows](https://github.com/brunkstr/Stegstr/releases/latest/download/Stegstr-Windows.exe) · [Linux](https://github.com/brunkstr/Stegstr/releases/latest/download/Stegstr-Linux.deb) / [AppImage](https://github.com/brunkstr/Stegstr/releases/latest/download/Stegstr-Linux.AppImage)

See [Releases](https://github.com/brunkstr/Stegstr/releases) for other builds and Android.

### Command-line interface (CLI)

Node 18+ only. `dist-cli/stegstr.mjs` is committed and self-contained, with no
runtime dependencies, so it runs straight from a bare clone with no `npm
install` and no compiler. That matters for offline or network-isolated build
environments, where installing dependencies is not an option:

```bash
node dist-cli/stegstr.mjs selftest
node dist-cli/stegstr.mjs embed cover.jpg -o out.jpg --payload "hello"
node dist-cli/stegstr.mjs detect out.jpg --json
node dist-cli/stegstr.mjs modes --json
```

To rebuild it from source: `npm install && npm run build:cli`.

Every command accepts `--json` and returns meaningful exit codes (`0` success,
`1` nothing found, `2` bad usage), so it drives cleanly from a script or an
agent. `embed` decodes its own output before reporting success.

### Does it really survive WhatsApp?

Check with your own phone rather than taking it on trust:

```bash
npm run testkit                    # writes 8 images to testkit/out/
# send them to yourself as normal photos, save the results into returned/
npm run testkit:verify returned/
```

## Build from source (full app)

Prerequisites: Node.js 18+, Rust (latest stable).

```bash
git clone https://github.com/brunkstr/Stegstr.git
cd Stegstr
npm install
npm run build:mac   # or build:win, build:linux
```

See the repo for platform-specific build deps (e.g. Xcode CLI tools, Visual Studio Build Tools, Linux dev packages).

## Links

- [Website](https://stegstr.com) — Downloads, getting started, wiki
- [Wiki / CLI docs](https://stegstr.com/wiki/cli.html) — Full CLI reference
- [Releases](https://github.com/brunkstr/Stegstr/releases)

## License

MIT
