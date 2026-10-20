# MagiskRenef — `par-primario` fork

A personal fork of [vichhka-git/magisk-renef](https://github.com/vichhka-git/magisk-renef)
that ships a **prebuilt** module instead of downloading the upstream release at build time.

Everything else in this repo — `base/`, `build.py`, `main.py`, the CI workflow — is
upstream, untouched. The upstream README is kept verbatim at
[`README-UPSTREAM.md`](README-UPSTREAM.md).

> **Branch layout**
> `par-primario` (this branch, **default**) — the fork: prebuilt binaries + this README.
> `main` — pristine mirror of upstream, for `git pull` / rebasing.

---

## Why this fork exists

The upstream module packages whatever the engine's **latest release** is — currently
`v0.3.6` of [`ahmeth4n/renef`](https://github.com/ahmeth4n/renef). After that tag the
engine replaced its injector, and the replacement changes how `-p` behaves:

| | engine `v0.3.6` | engine after `v0.3.6` (what this fork ships) |
|---|---|---|
| injection | patches `malloc` in the target | `ptrace` + `__loader_dlopen` in `linker64` — no instruction writes into code memory |
| `-p` / `--pause` | releases the app `on first exec` | **freezes the main thread**, releases **on script delivery** |

The second row is the reason. With `v0.3.6` binaries, `-p` does not hold the app long
enough for a script to land before the app's own code runs, so the spawn gate is not
usable. The post-`v0.3.6` build freezes the main thread and hands control back only when
the script arrives — which is what makes `-s <app> -p -l script.lua` work.

**There is no patch in this fork.** The binaries are an **unmodified build of upstream
`ahmeth4n/renef`** at a commit after the `v0.3.6` tag. The spawn-gate strings in them
appear verbatim in upstream `src/`:

```
src/librenef/cmd/cmd_spawn.cpp:199   [spawn-gate] main thread frozen (pid=
src/librenef/cmd/cmd_load.cpp:111    [spawn-gate] Script delivered, resuming (pid=
```

Upstream `main` at `cffcd5f` is 6 commits ahead of tag `v0.3.6`. The only difference from
the released `MagiskRenef-v0.3.6.zip` is **which upstream commit was compiled**.
Full details, hashes and reproduction steps: [`PROVENANCE.md`](PROVENANCE.md).

---

## Requirements

- Rooted Android device — **Magisk**, **KernelSU** (or KernelSU Next), or **APatch**
- **ARM64 only** — renef has no 32-bit or x86 Android build
- Android 10+ (tested on newer; see *SELinux* below)

## Install

1. Grab `dist/MagiskRenef-v0.3.6-par-primario.zip` from this repo.
2. Verify it:
   ```bash
   shasum -a 256 MagiskRenef-v0.3.6-par-primario.zip
   # 1bdeb3b08257a44cf4bb2a0583d8b1048584abb601221dbc642626e9e2311305
   ```
3. Install the zip through your root manager (Magisk / KernelSU / APatch).
4. Reboot.

> Do **not** install `MagiskRenef-v0.3.6.zip` from the upstream releases if you want the
> spawn gate — that one carries the `v0.3.6` engine binaries.

## Verify after boot

```bash
# the server must be up
adb shell su -c 'pidof renef_server'

# and it must be *this* build
adb shell su -c 'sha256sum /system/bin/renef_server'
# ac60155bef5ef7c0045ba2b1d1aa1d00039ecdc75fe500763368a6d25684eae9

adb shell su -c 'cat /data/local/tmp/renef_server.log'
```

In your root manager's module list, the description flips to one of:

- `✅ (running, UDS)` — healthy
- `❌ (failed)` — see the log above

---

## Usage

`renef_server` starts automatically at boot and listens on the **abstract UDS**
`@com.android.internal.os.RuntimeInit`. The client talks to it over ADB, so keep the
device connected.

### Client flags

```
-d, --device <id>     Device ID (optional)
-l, --load <script>   Load and execute Lua script
-a, --attach <pid>    Attach to process by PID
-s, --spawn <app>     Spawn application
-g, --gadget <pid>    Gadget mode: connect directly to injected agent (no server)
    --hook <type>     Hook type: trampoline (default) or pltgot
    --local           Local mode: connect via UDS (for Termux / on-device)
-m, --mode <mode>     Interface mode: v/view (TUI) [default: CLI]
-w, --watch           Auto-watch hook output after loading script
-p, --pause           Freeze app after spawn until script is loaded (spawn gate)
-v, --verbose         Enable verbose mode (show agent debug logs)
-h, --help            Show this help
```

Note that `-p` is a **modifier for `-s`**, not a target of its own.

### Spawn with the gate (the reason for this fork)

```bash
renef -s com.example.app -p -l your-script.lua -w
```

The app is frozen on its main thread until the script is delivered, then released.

### Attach to an already-running app

```bash
adb shell su -c 'pidof com.example.app'     # get the PID
renef -a <PID> -l your-script.lua
```

### On-device (no host client, e.g. Termux)

```bash
renef --local -s com.example.app
```

### If `-s` fails to launch the app

Some hardened apps set `android:exported="false"` on every activity; spawn goes through
`monkey` and cannot start them. Open the app by hand and use `-a`.

---

## The module's scripts

Nothing here is fork-specific — it is the upstream module. What each file does:

| file | what it does |
|---|---|
| `base/service.sh` | runs at late boot: `setenforce 0`, copies `libagent.so` to `/data/local/tmp/libagent.so`, tags it `u:object_r:app_data_file:s0`, then starts `renef_server` under `setsid` and logs to `/data/local/tmp/renef_server.log` |
| `base/action.sh` | the **action button** — toggles the server (kills it if up, starts it if down) and logs to `$MODPATH/logs/action.log` |
| `base/customize.sh` | install-time checks: ARM64 only, `module.prop` + both binaries present, sets SELinux contexts |
| `base/utils.sh` | shared helpers: `wait_for_boot`, `check_renef_is_up` |

**Running the action button:** tap *Action* on the module card in KernelSU / Magisk. It is
the supported way to start or stop `renef_server` without rebooting.

**By hand** (verified working):

```bash
adb shell su -c 'pkill -9 -f renef_server'
adb shell su -c 'setsid /system/bin/renef_server >/data/local/tmp/renef_server.log 2>&1 &'
```

## Troubleshooting

| symptom | cause | fix |
|---|---|---|
| `Failed to find libc base` | SELinux enforcing | `adb shell su -c setenforce 0`, then retry |
| `failed to start` in the module list | server died or was never root | check `/data/local/tmp/renef_server.log`; confirm `Uid: 0 0 0 0` in `/proc/<pid>/status` |
| injection does nothing, no error | the server answering is not root | restart it from a root context (see above or the action button) |
| `-s` cannot launch the app | `android:exported="false"` activities | use `-a` on a manually-opened app |
| wrong `renef_server` sha256 | you installed the upstream `v0.3.6` zip | reinstall `-par-primario.zip` |

---

## Binaries in this repo

| path | sha256 | note |
|---|---|---|
| `prebuilt/android-arm64/renef_server` | `ac60155bef5ef7c0045ba2b1d1aa1d00039ecdc75fe500763368a6d25684eae9` | **primary** — proven in use; exact commit undetermined |
| `prebuilt/android-arm64/libagent.so` | `e8ba6d26b5cd9b1c69499f81509e63781d52120d55fd278096f81ba1309ae807` | pairs with the above |
| `prebuilt/android-arm64/renef` | `4bcd61a601f45a1f836bd9a3b422a2112a9c17ad67598b2e1095860974dbffba` | ARM64 client, runs on-device |
| `prebuilt/android-arm64-cffcd5f/*` | `ec2eb9bb…` / `2f2d8a80…` | reserve pair, labelled after commit `cffcd5f` |
| `prebuilt/linux-x64/renef` | `61fdc0cdba9fe3667021661c2e2ee92e6786d0fc4109e09185483297cd90116e` | host client, x86_64 Linux |

**Do not mix pairs.** Client and server speak a private protocol; take
`renef_server`, `libagent.so` and the client from the *same* directory.

## Rebuilding

```bash
git clone https://github.com/ahmeth4n/renef.git
cd renef && git checkout cffcd5f
./build.sh          # -> build/renef_server, build/android/libagent.so
```

Then drop the two files into `base/system/bin/` and `base/system/lib64/` and repack —
either through `python build.py`, or by hand from the `base/` template.

The upstream build path (download the latest engine release, package it) still works
unchanged:

```bash
uv run python3 main.py
FORCE_RELEASE=1 uv run python3 main.py   # force a rebuild
```

> The CI workflow in `.github/workflows/` is **disabled** in this fork on the
> `par-primario` branch — upstream it publishes releases and pushes tags on every push to
> `main` and on a daily cron, which is not wanted here.

## Credits and licensing

- [renef](https://github.com/Ahmeth4n/renef) by Ahmeth4n — **Apache-2.0**
- [magisk-renef](https://github.com/vichhka-git/magisk-renef) by vichhka-git
- [magisk-frida](https://github.com/ViRb3/magisk-frida) by ViRb3 — structure inspiration

The engine binaries redistributed here are Apache-2.0; upstream's `LICENSE` and this
attribution must be preserved if you pass them on.

The module repository (`vichhka-git/magisk-renef`) ships **no licence file**, so by
default its own code is all-rights-reserved — this fork redistributes it unmodified for
personal use, and that is not a grant of rights over it.
