# Rocktor/tv

Home customization workspace for Rocktor Wang’s **Sony BRAVIA** (Android / Google TV).

This repo vendors [CleanYourTV](https://github.com/Gathin23/cleanyourtv) under [`cleanyourtv/`](cleanyourtv/) so we can debloat preloaded junk, install a clean Apple TV–style launcher, and keep changes reversible — all over Wi‑Fi ADB. No root; nothing is uninstalled.

## What’s here

| Path | Purpose |
|---|---|
| [`cleanyourtv/`](cleanyourtv/) | Full CleanYourTV tree (tools, launcher, Sony example lists, themes). MIT + OFL. |
| [`NOTICE`](NOTICE) | Upstream attribution and import pin. |
| [`docs/sony-setup.md`](docs/sony-setup.md) | Step-by-step Sony home-network setup (ADB → backup → safe debloat → launcher → restore). |

## License / attribution

CleanYourTV is **MIT** — Copyright (c) 2026 Gathin. The Outfit font in the launcher is **SIL OFL 1.1**. See [`cleanyourtv/LICENSE`](cleanyourtv/LICENSE) and [`NOTICE`](NOTICE). Keep those notices when you modify or redistribute.

## Cloud agents vs your living room

Cursor cloud agents **cannot** reach a home TV over ADB. All ADB steps must run on a laptop/desktop on the **same Wi‑Fi** as the TV. Use the scripts in `cleanyourtv/tools/` and the guide in [`docs/sony-setup.md`](docs/sony-setup.md).

## Quick start (on your home network)

```bash
git clone https://github.com/Rocktor/tv.git
cd tv/cleanyourtv
# Then follow docs/sony-setup.md (or cleanyourtv/README.md “Doing it by hand”)
```

## Syncing upstream

Imported with `git subtree` from `Gathin23/cleanyourtv` (see `NOTICE` for the pinned commit). To pull newer upstream into this prefix:

```bash
git fetch https://github.com/Gathin23/cleanyourtv.git main
git subtree pull --prefix=cleanyourtv https://github.com/Gathin23/cleanyourtv.git main --squash
```
