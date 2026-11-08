# Sony BRAVIA setup plan (CleanYourTV)

Concrete steps for Rocktor’s home Sony TV: developer mode is already on and the TV is on Wi‑Fi. Goal is a reversible debloat + CleanYourTV launcher, run only from a machine on the same LAN.

**Repo:** [Rocktor/tv](https://github.com/Rocktor/tv) → work inside `cleanyourtv/`  
**Upstream:** [Gathin23/cleanyourtv](https://github.com/Gathin23/cleanyourtv) (MIT)  
**Cloud agents cannot reach your TV.** Every ADB command below must run on a laptop/desktop on the same Wi‑Fi (or Ethernet that can talk to the TV’s subnet — avoid guest / AP-isolation SSIDs).

---

## 0. Prerequisites (computer on home Wi‑Fi)

| Need | Why |
|---|---|
| Same Wi‑Fi / LAN as the TV | Network ADB does not traverse the internet or Cursor cloud |
| **Python 3** | `cleanyourtv/tools/*` |
| **JDK 17–22** (full JDK, includes `jlink`) | Build the launcher; JDK 23+ breaks Gradle 8.9 |
| **Android SDK** platform **34** + build-tools **34.x** | Launcher build (or use Docker — see below) |
| Optional: Google **platform-tools** (`adb`) | Required if the TV only offers Wireless debugging pairing (common on newer Sony / Android 14) |
| Optional: **Docker** | `tools/docker-build.sh` builds the APK without a local SDK |

Clone:

```bash
git clone https://github.com/Rocktor/tv.git
cd tv/cleanyourtv
python3 -m venv tools/venv
tools/venv/bin/pip install -r tools/requirements.txt
```

macOS tip if nothing is installed yet:

```bash
brew install openjdk@17 android-commandlinetools
sdkmanager "platforms;android-34" "build-tools;34.0.0"
```

---

## 1. On the TV — finish debugging (dev mode already on)

Menus vary by firmware (Android TV vs Google TV, region, Android 12 vs 14).

### Typical Google TV / modern BRAVIA path

1. Settings → **System** → **Developer options** (already unlocked).
2. Enable **USB debugging** / **ADB debugging**.
3. If present, also enable **Network debugging** (shows IP:port, often `:5555`).
4. **Sony / Android 14 quirk:** some BRAVIA units only expose **Wireless debugging** (pairing code), not a sticky `:5555` port. Use that path (section 2B).
5. Keep the TV **awake** and note its IPv4 (Settings → Network / About, or your router’s client list).

### Older Android TV path

Settings → **Device preferences** → **About** (Build already pressed 7×) → Developer options → USB / network debugging on.

### Sony-specific callouts

| Topic | What to expect |
|---|---|
| Menu labels | “USB debugging”, “ADB debugging”, “Network debugging”, or only “Wireless debugging” |
| Port 5555 | Older firmware: open while network debugging is on. Newer (esp. Google TV 14 / AE2-class): pair Wireless debugging, then optionally `adb tcpip 5555` (resets after reboot) |
| Ethernet-only on some Android 14 Sony builds | Wireless debugging may require Wi‑Fi; Ethernet alone can disable it. Prefer Wi‑Fi for the ADB session (or Wi‑Fi + `adb tcpip 5555` then switch if needed) |
| China firmware (`_CN`, Dangbei home) | Stock Home may be `com.dangbei.TVHomeLauncher`, not Google `launcherx`. See `docs/devices/sony-x9000h-cn.md` and pass `STOCK_LAUNCHER=none` if the install script won’t recognize Home |
| Keep for HDMI / tuner / DRM | Do **not** disable `com.sony.dtv.tvinput.*`, `com.sony.dtv.tvapi`, `com.sony.dtv.sonysystemservice`, `com.sony.dtv.sonyvendorservice`, `com.sony.dtv.settingsprovider`, system update packages, partner DRM/enclave packages, or `com.mediatek.*` |

---

## 2. Connect ADB from the home computer

Export your TV IP (example `192.168.1.50` — replace everywhere):

```bash
export TV_HOST=192.168.1.50
```

### 2A. Classic network ADB (port 5555)

```bash
# Optional: prove stock adb sees the TV
adb connect "$TV_HOST:5555"
# Accept “Allow USB debugging?” on the TV → Always allow

cd /path/to/tv/cleanyourtv
tools/venv/bin/python -I tools/tv.py --probe
```

### 2B. Wireless debugging + pairing (Sony Android 14 / AE2-style)

On the TV: Developer options → **Wireless debugging** → **Pair device with pairing code** (note IP, **pairing** port, 6-digit code). Separately note the **connection** port on the Wireless debugging main screen (different from the pairing port).

```bash
adb pair <TV_IP>:<PAIRING_PORT>    # enter the 6-digit code when prompted
adb connect <TV_IP>:<CONNECTION_PORT>

# Option 1 — open plaintext 5555 until next reboot (tools/tv.py default):
adb -s <TV_IP>:<CONNECTION_PORT> tcpip 5555
export TV_HOST=<TV_IP>             # tools talk to :5555

# Option 2 — keep TLS Wireless debugging only:
export TV_ADB_BINARY="$(command -v adb)"
export TV_HOST=<TV_IP>:<CONNECTION_PORT>
```

Details: `cleanyourtv/tools/native-adb.md` and `cleanyourtv/docs/TROUBLESHOOTING.md`.

Accept the RSA / debugging prompt on the TV before continuing.

---

## 3. Probe and backup (do this before any disable)

```bash
cd tv/cleanyourtv
mkdir -p ~/cleanyourtv-backup
tools/venv/bin/python -I tools/tv.py --probe | tee ~/cleanyourtv-backup/probe.txt
tools/venv/bin/python -I tools/tv.py 'pm list packages' | tee ~/cleanyourtv-backup/packages.txt
tools/venv/bin/python -I tools/tv.py --screenshot ~/cleanyourtv-backup/before.png
```

Identify model / platform (`BRAVIA_VH1`, `BRAVIA_AE2`, …) and Android version from the probe. Match a list under `examples/sony-*` (49 Sony folders upstream) or start from the closest platform (e.g. `examples/sony-bravia-ae2/` for AE2 / Google TV 14).

`tools/disable.sh` also writes a restore record under `~/.cleanyourtv/` keyed by `TV_HOST` — keep the same `TV_HOST` string for undo.

---

## 4. First **safe** debloat (small batch)

1. Copy a close Sony list locally so you can edit without fighting upstream:

```bash
mkdir -p ../custom
cp -R examples/sony-bravia-ae2 ../custom/sony-mine   # or your matched folder
```

2. Open `../custom/sony-mine/debloat.txt`. **Comment out** anything you use (YouTube Music, casting, voice, demo apps you want). Prefer a short first pass: Samba TV / ACR, Sony promo / Select / tab, feedback, obvious analytics — leave HDMI, settings, update, and DRM lines alone (they should already be absent from good lists).

3. Apply:

```bash
export TV_HOST=192.168.1.50
tools/disable.sh ../custom/sony-mine/debloat.txt
```

Nothing is uninstalled — only `pm disable-user`. If something breaks, `pm enable <package>` or run restore (section 7).

4. Reboot-check optional: after a power cycle, re-run `tools/tv.py 'pm list packages -d'` and compare.

On Google TV, also plan to disable `com.google.android.tungsten.setupwraith` before / with the stock launcher swap (it can steal HOME). Good Sony lists already include it.

---

## 5. Build and install the launcher

Still on the home computer, from `cleanyourtv/`:

```bash
export TV_HOST=192.168.1.50
# Local JDK + SDK:
launcher/build-and-install.sh

# Or Docker build, then install only:
tools/docker-build.sh
SKIP_BUILD=1 launcher/build-and-install.sh
```

The script installs `com.gathin.tvhome`, sets it as Home, and rolls back if Home does not stick.

**Sony Home variants**

| Stock Home | What to pass |
|---|---|
| Google TV `com.google.android.apps.tv.launcherx` | Default path; restore with that package as stock arg |
| Classic `com.google.android.tvlauncher` / leanback | Usually auto-detected |
| China Dangbei `com.dangbei.TVHomeLauncher` | `STOCK_LAUNCHER=none` then set Home manually if needed (see `docs/devices/sony-x9000h-cn.md`) |

After install: press Home — you should land on the CleanYourTV dock. Pull a screenshot:

```bash
tools/venv/bin/python -I tools/tv.py --screenshot ~/cleanyourtv-backup/after-home.png
```

---

## 6. Customize

| Goal | Command / place |
|---|---|
| Theme (colors, dock, wallpaper) | `tools/theme.sh themes/aurora` (or another folder under `themes/`) |
| Reset theme | `tools/theme.sh --reset` |
| Own photos as wallpaper | `python3 tools/wallpaper_prep.py ~/Pictures/trip mytheme/wallpapers` then push as a theme |
| Dock / columns | Theme `config.json` or launcher settings documented in upstream README |
| Add / remove packages later | Edit your `debloat.txt`, re-run `disable.sh` (record grows safely) or `pm enable` |
| After a firmware OTA | `tools/venv/bin/python -I tools/check.py ../custom/sony-mine/debloat.txt` then `--fix` if needed |

Keep custom lists and themes under `Rocktor/tv` (e.g. top-level `custom/`) so they survive upstream subtree pulls.

---

## 7. Restore (one-liner undo)

```bash
export TV_HOST=192.168.1.50
# Android TV-style:
tools/restore.sh ../custom/sony-mine/debloat.txt
# Google TV — also pass stock launcher:
tools/restore.sh ../custom/sony-mine/debloat.txt com.google.android.apps.tv.launcherx
```

Restore replays the `~/.cleanyourtv/` record (enabled/disabled state + who owned Home). It leaves the new launcher APK installed; remove it from Settings → Apps if you want it gone. Without a record, restore falls back to enabling the whole list.

Windows: `tools\restore.ps1` (see upstream README).

---

## 8. Lock down when finished

1. Turn **Wireless debugging** / **Network debugging** / **USB debugging** **off** on the TV when you are not actively changing it.
2. Revoke unused USB debugging authorizations in Developer options.
3. Confirm the port is closed: `python3 tools/adb-closed.py <TV_IP> 5555` (or your Wireless debugging port).

The custom launcher keeps working with debugging off.

---

## Suggested first session checklist

1. [ ] Laptop on same Wi‑Fi; clone `Rocktor/tv`; create `tools/venv`
2. [ ] Confirm Developer options debugging toggles; note IP
3. [ ] `adb` connect / pair; accept TV prompt
4. [ ] `tools/tv.py --probe` + package list + before screenshot → `~/cleanyourtv-backup/`
5. [ ] Match `examples/sony-*`; copy to `custom/sony-mine`; trim list
6. [ ] `tools/disable.sh` on the trimmed list
7. [ ] `launcher/build-and-install.sh` (or Docker path)
8. [ ] Verify Home; optional theme
9. [ ] Know the restore command; turn debugging off

**Exact next action for Rocktor:** On a home-network machine, clone the PR branch (or `main` once merged), `cd cleanyourtv`, create the Python venv, turn on Network/Wireless debugging on the Sony, set `TV_HOST` to the TV’s IP, and run `tools/venv/bin/python -I tools/tv.py --probe` until it prints model/firmware — then proceed from section 3.
