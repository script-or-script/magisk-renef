# Provenance — `MagiskRenef-v0.3.6-par-primario.zip`

This branch carries **no patch**. It vendors a prebuilt module zip whose engine
binaries were compiled from upstream [`ahmeth4n/renef`](https://github.com/ahmeth4n/renef)
at a commit **after** the `v0.3.6` tag, together with the binaries themselves.

## Why a post-`v0.3.6` build is needed

`v0.3.6` is the tag this module upstream packages — `build.py` downloads
`renef-v0.3.6-android-arm64.tar.gz` from the engine's releases. Between `v0.3.6`
and `cffcd5f` the injector was replaced:

| | `v0.3.6` | post-`v0.3.6` |
|---|---|---|
| injection | patches `malloc` | `ptrace` + `__loader_dlopen` in `linker64` — no instruction writes into code memory |
| spawn gate (`-p`) | releases `on first exec` | freezes the **main thread**, releases **on script delivery** |

The spawn-gate behaviour is the operative difference: with the `v0.3.6` binaries
the window a `-p` spawn opens does not exist.

## This is unmodified upstream code

The strings in the prebuilt binaries appear **verbatim** in upstream `src/`:

```
src/librenef/cmd/cmd_spawn.cpp:199   [spawn-gate] main thread frozen (pid=
src/librenef/cmd/cmd_load.cpp:111    [spawn-gate] Script delivered, resuming (pid=
```

Upstream `main` at `cffcd5f` is **6 commits ahead of** tag `v0.3.6`. Nothing here
is a private or reverse-engineered change — the only difference from the released
`MagiskRenef-v0.3.6.zip` is *which upstream commit was compiled*.

## Binaries

| path | sha256 | size | note |
|---|---|---|---|
| `prebuilt/android-arm64/renef_server` | `ac60155bef5ef7c0045ba2b1d1aa1d00039ecdc75fe500763368a6d25684eae9` | 10138360 | **primary pair** — post-`v0.3.6`, exact commit not determined; proven on hardware |
| `prebuilt/android-arm64/libagent.so` | `e8ba6d26b5cd9b1c69499f81509e63781d52120d55fd278096f81ba1309ae807` | 9645648 | pairs with the above |
| `prebuilt/android-arm64/renef` | `4bcd61a601f45a1f836bd9a3b422a2112a9c17ad67598b2e1095860974dbffba` | 10394008 | ARM64 client, runs on-device |
| `prebuilt/android-arm64-cffcd5f/renef_server` | `ec2eb9bbfe433af3f40e05a6f8c956449b7c2239c17808882c5e61af15883409` | 10203016 | reserve pair, labelled after commit `cffcd5f` |
| `prebuilt/android-arm64-cffcd5f/libagent.so` | `2f2d8a800376b1f463a8139a47806ee5f699e4aa34499e368bf17812f483ee53` | 9646352 | pairs with the above |
| `prebuilt/linux-x64/renef` | `61fdc0cdba9fe3667021661c2e2ee92e6786d0fc4109e09185483297cd90116e` | 11219864 | host client, x86_64 Linux |
| `dist/MagiskRenef-v0.3.6-par-primario.zip` | `1bdeb3b08257a44cf4bb2a0583d8b1048584abb601221dbc642626e9e2311305` | 2744186 | module zip: `base/` scripts + the two primary binaries |

**Do not mix the pairs.** Client and server speak a private protocol; a mismatched
set fails. Take `renef_server` + `libagent.so` + client from the *same* directory.

The `cffcd5f` label on the reserve pair comes from the packaging provenance, not
from an independent rebuild — it has not been recompiled and byte-compared here.
The primary pair carries the post-`v0.3.6` spawn-gate strings but no embedded
commit marker, so its exact commit is unknown.

## Reproducing

```bash
git clone https://github.com/ahmeth4n/renef.git
cd renef && git checkout cffcd5f
./build.sh                 # -> build/renef_server, build/android/libagent.so
```

Then drop the two files into `base/system/bin/` and `base/system/lib64/` and run
`python build.py`, or repack by hand from the `base/` template in this repo.

## Licensing

The engine (`ahmeth4n/renef`) is **Apache-2.0**; keep its `LICENSE` and state
modifications when redistributing. This module repository ships **no licence
file**, so its own code is all-rights-reserved by default — keep this fork
private unless that changes.
