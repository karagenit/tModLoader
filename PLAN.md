# PLAN: tModLoader 1.4.6-alpha (unofficial port to vanilla Terraria 1.4.6.1)

Branch `1.4.6-alpha` in fork `karagenit/tModLoader` (`origin`), based on upstream `tModLoader/tModLoader` branch `1.4.5` (`upstream` remote) at 9d19c0f61.
This is the same commit as the Steam `1.4.5-dev` build installed on this Mac. Read `AGENTS.md` first (fork rules). Keep the Status log at the bottom current.

## Goal

Build and run a local tModLoader on vanilla Terraria **1.4.6.1** on this Mac, so that the user's current vanilla saves can be played modded with
the TerrariaTimeLimit mod (repo `~/Documents/terraria-time-limit`).

Done when:
1. setup completes, and all three patch layers apply with no new rejects (the 11 inherited upstream `.rej` files are tracked separately, see Phase 0).
2. tModLoader builds and launches from its dev folder, and reaches the main menu with mods loaded.
3. **Copies** of the save-version-329 player `JB.plr` and world `The_DLC_of_No_Remorse.wld` load and play. Saving and reloading round-trips.
4. TerrariaTimeLimit builds against this tML and loads.

Not a goal: cross-play, Workshop/publishing, upstreaming (if that ever happens, the user redoes the work by hand, see AGENTS.md), Windows/Linux.

## Background (verified 2026-11-26)

- Upstream tML `1.4.5` is built on vanilla **1.4.5.8** (`Main.curRelease` = **326**). Vanilla 1.4.6.1 has curRelease **329**.
  tML 1.4.5 shows 329 saves as "(LaterVersion)" and can't load them. This is the reason for the port.
- Upstream tracking issue: tModLoader/tModLoader#5607. "Update patches on the 1.4.5 branch" is listed as TODO Later, not started. The branch may get renamed.
- 1.4.6 changes (`changelog.txt` in the Steam Terraria folder): crossplay (redesigned PC multiplayer UI, Steam friends join/invite, LAN discovery,
  dedicated server lists), the new "creepy crawlies" seed (and Arachnophobia seed changes), item stacking rules in the world, balance changes and bug fixes.
  Expect patch conflicts to concentrate in networking (Netplay, NetMessage, MessageBuffer), Social/Steam, the multiplayer menu UI, and world gen/seed
  flags plus WorldFile save code. Crossplay UI that conflicts with tML can be stubbed or disabled rather than ported.
- Read curRelease from any assembly with the tool in `/tmp/ttl-constreader`
  (`dotnet /tmp/ttl-constreader/out/constreader.dll <dll|exe> Terraria.Main curRelease`). /tmp may have been cleared: it's about 25 lines using System.Reflection.Metadata.
- Read a save's version: see `~/Documents/terraria-time-limit/.agents/notes/env/terraria-versions-saves.md`.

## Environment

- Intel Mac, macOS 13.4. .NET 10 SDK 10.0.404 at `~/.dotnet`, not on PATH: `export PATH="$HOME/.dotnet:$PATH"`. The system `dotnet` is 8.0 only.
- Steam vanilla Terraria 1.4.6.1 (Mac build): `~/Library/Application Support/Steam/steamapps/common/Terraria/Terraria.app/Contents/Resources`
  (this is the TerrariaSteamDirectory setup uses, auto-detected).
- Steam tML 1.4.5-dev: `.../steamapps/common/tModLoader` (save dir `~/Library/Application Support/Terraria/tModLoader-dev`).
- Existing agent notes from the mod project (read the relevant ones, don't duplicate them here):
  `~/Documents/terraria-time-limit/.agents/notes/`. In particular `env/tmodloader-145-dev-macos.md` (Steam client version, GPU switching crash,
  FNA stale display id patch, Steam "Files Corrupt" re-verify gotcha), `env/tmodloader-macos.md`, `env/dotnet-sdk.md`, `tmodloader/api-facts.md`.
- This repo's own notes go in `.agents/notes/` (add `.agents/` to `.git/info/exclude`, never commit it).

## Pipeline (upstream setup tooling, run via `./setup-cli.sh <command>`)

`setup-cli.sh` inits submodules (FNA, setup/PatchReviewer) on first run, then runs `dotnet run --project setup/CLI/Setup.CLI.csproj -- <args>`.

| command | from | patches | to |
|---|---|---|---|
| `decompile` | Windows Terraria.exe + TerrariaServer.exe | none | `src/decompiled` |
| `patch terraria` | src/decompiled | patches/Terraria | src/Terraria |
| `patch netcore` | src/Terraria | patches/TerrariaNetCore | src/TerrariaNetCore |
| `patch tml` | src/TerrariaNetCore | patches/tModLoader | src/tModLoader |

- `setup` = decompile + `regen-source` (the 3 patch steps). Useful flags: `-f/--no-prompts`, `--plain-progress`, `--terraria-steam-dir <dir>`,
  `--tml-dev-steam-dir <dir>`, and on patch commands `-m EXACT|OFFSET|FUZZY`. setup/regen-source silently downgrade FUZZY to OFFSET,
  so run the `patch <layer> -m FUZZY` commands individually.
- Results: `setup/logs/patch.log` (per file exact/offset/fuzzy/failed). Rejects are `.rej` files (probably next to the file in src/; confirm).
  The PatchReviewer GUI isn't usable from the CLI or on macOS. Review fuzzy hunks by hand, because fuzzy can apply a hunk in the wrong place.
- After fixing a layer in `src/<layer>`, regenerate its patches with `diff terraria|netcore|tml`. It only diffs files newer than the cutoff
  stored in `setup/user.settings`, and a patch run resets that cutoff. So run patch -> fix -> diff per layer, in order.
  Then re-run the next layer's patch, because it builds on the previous layer's output.
- `terraria-version <exe>` prints an exe's assembly version.

### Version pins to change (1.4.5.8 -> 1.4.6.1)
- `setup/Core/TerrariaDecompileExecutableProvider.cs:12-13`: ClientVersion/ServerVersion.
- `setup/Core/SecretKeyProvider.cs:8`: PreviousKeyClientVersion (secret commands only, but keep consistent).
- `<Version>1.4.5.8</Version>` context lines in `patches/Terraria/Terraria/Terraria.csproj.patch` and `patches/TerrariaNetCore/Terraria/Terraria.csproj.patch`.
  These will reject; fix by hand.
- `patches/tModLoader/Terraria/ModLoader/Engine/InstallVerifier.cs` TerrariaVersion (already stale at 1.4.5.6 upstream, so decide whether it matters).
- curRelease/versionNumber come from the decompiled vanilla source, not from tML. Watch the `Main.cs.patch` hunks around them.

### Getting the 1.4.6.1 Windows executables
Setup only decompiles the **Windows** build. Normally it decrypts `setup/SecretAssets/Terraria_v1.4.5.8_win.exe.enc` with a key derived from the
SHA256 of the user's own Terraria exe. That key can't be derived from 1.4.6.1, and there's no 1.4.6.1 `.enc`. Bypass: if
`<TerrariaSteamDirectory>/Terraria_v<ver>_win.exe` and `TerrariaServer_v<ver>_win.exe` already exist, setup uses them as-is
(no decryption, no version check, so verify them with `terraria-version`).
- Client: the Windows depot of Steam app 105600. In the Steam console (`steam://open/console`), run `app_info_print 105600` to find the depot
  whose oslist is windows (believed to be 105601, unverified). Then `download_depot 105600 <depot>` (no manifest = latest). Steam prints the
  output path. The user must type this in Steam's console; the agent can't.
- Server: `https://terraria.org/api/download/pc-dedicated-server/terraria-server-1461.zip` (believed to contain `1461/Windows/TerrariaServer.exe`).
  Download into its own fresh directory (untrusted-download rule).
- Copy both into Terraria.app/Contents/Resources with the versioned names above. Steam may delete these extra files on a Terraria
  update or verify, so keep originals elsewhere (e.g. `~/Documents/tml-146-vanilla/`).

### Built output and running
- A build of src/tModLoader copies output plus `release_extras` into the tML dev dir, which defaults to
  `~/Library/Application Support/Steam/steamapps/common/tModLoaderDev`. Run with `cd <devdir> && dotnet tModLoader.dll`.
  Unverified: whether the build supplies a dotnet runtime, Steam init when launched outside Steam, and DYLD_LIBRARY_PATH for natives.
- **Save dir collision:** a local dev build defaults to the same `tModLoader-dev` save dir as the Steam 1.4.5-dev install. Always launch with
  `-tmlsavedirectory "$HOME/Library/Application Support/Terraria/tModLoader-146alpha"` (or similar), so 1.4.6 (v329) saves never land where Steam
  1.4.5-dev can't read them.
- FNA: the `FNA` submodule is the same sha (48d51381) as the patched clone in `~/Documents/tml-fna-patch/`. The local build needs the
  stale-display-id fix too, or automatic GPU switching crashes it. Choose one: apply `stale-display-id.patch` to the submodule working tree
  (uncommitted), fork FNA and bump the submodule, or copy `~/Documents/tml-fna-patch/out-patched/FNA.dll` over the built one.

## Phases

### Phase 0: toolchain and inputs
1. `export PATH="$HOME/.dotnet:$PATH"`, then `./setup-cli.sh --help` (inits submodules, builds the CLI). Fix any macOS toolchain issues first.
2. Get the 1.4.6.1 Windows Terraria.exe + TerrariaServer.exe (above). Verify both versions.
3. Optional baseline: run the unmodified pipeline against 1.4.5.8 to separate "macOS tooling problems" and the 11 inherited upstream `.rej` files
   (under patches/tModLoader/Terraria/: Chest, Lang, Main, Player, Projectile, Recipe, NetClientSocialModule, ChatManager, ChestUI, ItemSlot,
   WorldGen) from new 1.4.6 rejects. Inputs: the 1.4.5.8 server is free (`terraria-server-1458.zip`), and the 1.4.5.8 client needs
   `download_depot` with the old manifest id (from SteamDB, via the user's browser).
   Possibly skip it and just treat the 11 files as known.

### Phase 1: decompile 1.4.6.1
Bump the version pins, then run `./setup-cli.sh decompile -f --plain-progress`. Commit the pin changes.

### Phase 2: scout (measure before fixing)
Run `patch terraria -m FUZZY`, then `netcore`, then `tml`, each layer on top of whatever the previous produced, even with rejects.
Record the failed/fuzzy counts per layer and the list of rejected files from patch.log in the Status log. This sizes the work.
Check in with the user before starting the big fix loop.
Also useful for understanding what vanilla changed: decompile the free 1.4.5.8 and 1.4.6.1 *servers* (`decompile --server-only` ->
`src/decompiled_server`, or ilspycmd into scratch dirs) and diff them. The server shares world/save/network code, so this gives a cheap vanilla
1.4.5.8 -> 1.4.6.1 diff for WorldFile/Player/NetMessage without the old client exe.

### Phase 3: fix rejects, layer by layer
For each layer in order (terraria -> netcore -> tml): fix the `.rej` files in `src/<layer>`, review fuzzy hunks, get `src/<layer>` compiling
where practical, run `diff <layer>`, commit (`patches/` only), then re-run the next layer's patch. Commit often, in small batches (upstream did the
1.4.5 port in "Batch N (X broken patches remain)" commits). Committing `.rej` files mid-port is acceptable, as upstream did.
Prefer the simplest port. Where crossplay rewrote something tML heavily patched, keep vanilla behaviour on the paths tML needs and
disable the crossplay-only parts. Note these decisions in the Status log.

### Phase 4: save format
- With vanilla at 329, tML's patched WorldFile/Player load/save must read and write any new 1.4.6 fields (new seed flags etc.).
  Use the server diff from Phase 2 to find them.
- Check tML's own version gates (TileEntity.TML.cs, MessageBuffer/NetMessage version checks, ModLoader save sidecar files .twld/.tplr).

### Phase 5: build, run, test
- Build src/tModLoader (.NET 10) and handle FNA (above). Launch with the separate save dir.
- Reach the main menu, and check client.log (in `<devdir>/tModLoader-Logs/`).
- **Back up** the user's vanilla `~/Library/Application Support/Terraria/{Players,Worlds}` first, then load *copies* of JB.plr and
  The_DLC_of_No_Remorse.wld into the separate save dir. Play, save, reload. The user must do the in-game testing (AGENTS.md).

### Phase 6: the mod
Build TerrariaTimeLimit against the dev dir's `tMLMod.targets`. Its `src/tModLoader.targets` currently imports from the Steam tML install path,
so it needs a switch or property; see that repo's `.agents/notes/project/build-install.md`. Then load it in-game.

## Risks / open questions
- Whether upstream setup works on macOS at all (decompile via ICSharpCode, formatting, .NET Framework reference download). Phase 0 answers this.
- The size of the crossplay/networking conflict. Phase 2 answers this.
- Upstream keeps moving. Rebasing 1.4.6-alpha on later upstream 1.4.5 commits means re-resolving patch conflicts, so do it rarely, and only if needed.
  If upstream publishes its own 1.4.6 port, switch to it.
- Running the dev build outside Steam (Steam API init, launch scripts, runtime). Phase 5 answers this.

## Status log
- 2026-11-26: Fork created, branch 1.4.6-alpha at upstream 1.4.5 9d19c0f61. AGENTS.md fork section and this plan added. Nothing run yet.
- 2026-11-26: Decisions (user): crossplay is not needed. Stub/disable crossplay-only code (GNS, LAN discovery, server lists, new MP UI)
  wherever it conflicts, and keep tML's existing Steam/direct-IP multiplayer. Note: vanilla 1.4.6 ships a new native `libGNS_Wrapper.dylib`
  (osx/ next to libsteam_api) that tML doesn't bundle, which is another reason to stub. Skip the Phase 0 1.4.5.8 baseline and treat the 11 rejects as known.
  FNA: option (a), stale-display-id.patch applied to the FNA submodule working tree (uncommitted, `m FNA` in git status). It also adds
  `[FNA-PATCH]` log lines. Save dir: `~/Library/Application Support/Terraria/tModLoader-146alpha`. Commit to 1.4.6-alpha and push to `origin` freely.
  Fixing is delegated heavily to Haiku subagents, with high-level validation only.
- 2026-11-26: Phase 0.1 done. `./setup-cli.sh --help` inits submodules and builds the CLI fine on macOS (.NET 10.0.404). Waiting on the
  user's `download_depot` for the Windows client.
