# Fileditch Integration & Post-Download Extraction — Implementation Report

This replaces the earlier speculative analysis with what actually shipped: fileditch.com as the
third download source of Orbit Store (alongside Archive.org and Vikingfile) and the fully
automatic background flow **download → verify → extract → install → delete archive**, discovered
through dlpsgame.com. Verified against the source bundle, the published catalogue and the build
outputs.

**Result:** catalogue revision 15 (636 games, 724 options, 59 fileditch releases — 32 multi-volume
`.partN.rar` sets and 27 singles), console and host binaries build clean, all
syntax/type/format/catalogue checks green. Phase 1 shipped the single-volume `.rar` flow at
revision 14; phase 2 (§11) adds multipart sets, 7z/zip extraction and catalogue passwords.

---

## 1. Discovery: dlpsgame.com is the index, fileditch is the store

- dlpsgame.com publishes one post per game, naming the PlayStation title id
  (`PPSAxxxxx`) and mirroring the archive as a plain `[DLPSGAME.COM]-<title id>` file on
  fileditch.
- Crawl (`fd-crawl-results.json`): 83 posts → 59 with fileditch links → phase 1 kept **22 posts
  whose only live file is one single-volume `.rar`** and excluded the 36 multi-volume posts
  (`.partN.rar`) and the 1 post with a `.7z` mirror; dead links (none — fileditch files were all
  alive at crawl time). Phase 2 admits all 59 posts (§11).
- Every candidate was probed live through the fileditch **Status API**
  (`https://fileditchfiles.st/api/<user>/<hash>/<file>` → `{"status":true,"size":…}`), which
  provides both the existence proof and the exact byte size used for progress and verification.
- Archive layout (verified on a real download): one header+data encrypted, non-solid RAR5
  archive containing a wrapper folder `[DLPSGAME.COM]-<tid>/`, a `Note.txt` and a deeper
  `decrypted/` copy of the game. That double nesting is why extraction must lift the *shallowest*
  directory holding `sce_sys/` + `eboot.bin` (§5), and why the archive password
  `DLPSGAME.COM` is required (encrypted headers).
- fileditch cannot be queued as a plain HTTP transfer: the browser page sits behind a Cloudflare
  JS challenge and the real CDN route is **signed** (`md5`+`expires` or `exp`+`sig`) with a
  rotating host. The catalogue therefore stores the *page* URL and the actual route is captured
  from the console browser session — the same handoff model as Vikingfile, implemented
  independently (§3).

## 2. Catalogue integration

### Row shape (fileditch flavour)

```jsonc
{
  "id": "ppsa21402-folder",              // <titleid>-folder, < 24 chars
  "sourceId": "fileditch",
  "provider": "Fileditch",
  "format": "Folder",                    // archive-wrapped folder, extracted by Orbit
  "filename": "[DLPSGAME.COM]-PPSA21402.rar",
  "url": "https://fileditchfiles.st/<user>/<hash>/<file>",
  "browserUrl": "<same as url>",         // page == identity, capture supplies the route
  "sha256": "",                          // provider publishes no digest
  "verification": {                      // kind "file-metadata" (Status API)
    "metadataUrl": "https://fileditchfiles.st/api/…",
    "fileHash": "<hash segment>", "sizeBytes": …, "checkedAt": …, …
  },
  // merged game fields: cover/hero (Prosperopatches icon0/pic0 when the title page
  // exists, otherwise the dlpsgame post image kept as coverFallback/heroFallback)
}
```

No `metadataUrl` on new games: `fetch_metadata` only accepts playstation.com hosts, so a
missing PS-store page is simply a null-metadata game (genre/description/publisher are
null-safe in both frontends and already appear null in shipped rows).

### Tooling (all in `tools/`, stdlib-only Python)

| Step | Tool | What it does |
|---|---|---|
| Validate + gate | `catalog.py` | New `fileditch_identity` / `fileditch_page` / `fileditch_filename` (prefix, exactly one title id, single volume), `fileditch_status_url`, `fileditch_metadata_evidence`, a `verify_release` Status-API branch, and compile gates: `sourceId` ∈ {archive, vikingfile, fileditch}, `Folder` ⇔ fileditch only, `browserUrl` must be the fileditch page and equal `url`, suffix `.rar`, provider identity, evidence with per-provider expected `metadataUrl`/`fileHash` |
| Generate | `fd_catalog.py` | Reads the crawl, re-verifies each file live (1.2 s apart), fetches post artwork (`og:image`, measured for `wide` vs `ambient`), validates the full combined catalogue, then appends 22 releases + 21 games |
| Artwork | `artwork.py` | Prosperopatches title-page pass: 14 new games got official `icon0`/`pic0` art, 589 verified wide banners applied; dlpsgame images stay as `coverFallback`/`heroFallback` for the 13 titles without a title page |
| Publish | `publish_catalog.py --public-repo …` | Validates, bumps the revision, writes `catalogue.json` (legacy, Archive-only) + `catalogue-v2.json` + `catalog/{catalog,games,revisions}.json` |
| Embed | `embed.py` | Regenerates `build/generated/catalog.h` from the compiled catalogue |

`python3 tools/catalog.py check` reports **602 games, 687 validated options**; the embedded and
published feeds agree byte-for-byte.

## 3. Console backend — sources, provider boundary, browser capture

- `orbit.h`: `ORBIT_SOURCE_FILEDITCH 4U`; `sources.c` registers the third provider
  (`{"fileditch", "Fileditch", ready}`) and widens the selection message to three providers.
  Enabling still requires the existing source-notice acknowledgement.
- `providers.c` is the trust boundary:
  - `fileditch_page()` — strict page shape `https://fileditchfiles.st/<user alnum>/<20 hex>/<filename>`
    with no query/fragment, filename equal to the release filename, printable ASCII only.
  - `fileditch_cdn()` — accepts any https host only when the query is a **signed link**
    (`md5`+`expires`, or `exp`+`sig`) and the path is bound to the stored page path.
    CDN hosts rotate; the bound page path never does.
  - `provider_browser_supported()` — fileditch rows are browser-only: the page must validate and
    `url == browserUrl`. `provider_capture_url()` accepts exactly one captured candidate route.
- `browser.c` scans the opened page for the per-source prefix (`https://fileditchfiles.st/` vs
  `https://vikingfile.com/d/`), drives per-source intro/opening copy, and now reports `source`
  in `browser_status()` so every frontend names the right provider.
- `browser_match.c` matches candidates on the stable scheme only (fileditch hosts rotate),
  walking ASCII and UTF-16 byte strides, cutting at URL boundaries; each candidate must pass
  `provider_capture_url` before it is retained — cookies, credentials and storage signatures
  can never be captured.
- `api.c` strips `url`/`browserUrl` from the catalogue feed and derives `browserAvailable` /
  `directAvailable` from the provider functions, so fileditch rows surface as
  `browserAvailable: true, directAvailable: false` in both frontends.

## 4. Fully automatic post-download flow

`transfer.c`, after size + checksum verification of a fileditch job:

1. status → **`extracting`** (mutex-held, state saved),
2. `job_extract(job, dir, partial, err, cap)` (§5),
3. on `ORBIT_EXTRACT_OK` the `.part` archive is unlinked — the download is consumed;
   on `FAILED`/`CANCELLED` the archive is preserved so a resume never re-downloads,
4. status → `complete` (or `error` with the extraction message).

"Fully automatic" means no user interaction between the queue and the installed folder: the
worker performs extract → lift → patch → cleanup in the background while the UI shows
`Extracting`.

### `extract.c` — inspect → staging → lift → patch → cleanup

- `job_extracts()` — Phase-1 gate: single-file dlpsgame dumps only (`source == "fileditch"`).
- **Inspect**: archive present, staging destination on the same rooted filesystem, free space
  (`statvfs`) covers archive + extracted size + margin, before a byte is written.
- **Staging**: extraction goes into `<root>/.orbit-staging-<jobid>` — never directly into
  `homebrew/`, so a failed run cannot leave a half-installed game visible to the Library.
- **Extract**: vendored UnRAR (§6) with the archive password supplied through the
  `UCM_NEEDPASSWORD` callback; cancellation returns `-1` from `UCM_PROCESSDATA`, which unrar
  surfaces as `RARX_USERBREAK` → `ORBIT_EXTRACT_CANCELLED`.
- **Lift**: breadth-first walk (depth ≤ 4, `lstat` only — never follows a symlink out) finds the
  *shallowest* directory containing both `sce_sys/` and `eboot.bin` (the real game root inside
  the wrapper/`decrypted` nesting), which is renamed to `<root>/homebrew/<sanitized name>`.
- **Patch**: best-effort `sce_sys/params.json` → `"applicationDrmType":"standard"` via
  temp file + `fsync` + `rename`; a patch failure never fails the extraction.
- **Cleanup**: staging tree removed on every exit path.
- Console discipline mirrors ps5upload's payload: heap buffers for anything that grows, bounded
  path buffers (`EXTRACT_PATH_MAX`), no stack blobs.

Restart safety (`state.c`): a persisted `extracting` status is mapped to
`paused — "Interrupted. Resume to continue."`; the archive is still on disk, so resume continues
without redownloading.

## 5. Status flow across every surface

| Surface | `extracting` handling |
|---|---|
| `queue.c` | part of the *active* set (pause accepted, cancel aborts UnRAR via user break) |
| `state.c` | restart → `paused` + resume message |
| Web `types.ts` | status union member |
| Web `App.tsx` | counted as an active job (queue badge) |
| Web `collection.ts` | rank 1 (between downloading and verifying), label "Extracting" |
| Web `Downloads.tsx` / `styles.css` | active row, "Extracting archive…" meta, `.status.extracting` ink |
| TV `model.cpp` | `status_rank` 1, `job_active` true, `status_label` "Extracting" |
| TV `downloads.cpp` | blue status ink, "Extracting archive…" instead of a stale ETA |
| TV `model_test.cpp` | table + `job_active` assertions added |

## 6. Build system — vendored UnRAR

- `third_party/unrar` = **UnRAR 7.30 beta 1**, byte-identical to the upstream tarball
  (`diff -rq` clean). Licence recorded in `THIRD-PARTY-NOTICES.md` with `licenses/unrar.txt`;
  embedded third-party notices (Intel Slicing-by-8 CRC under BSD-3-Clause, public-domain
  AES/SHA-1/PPMII, BLAKE2sp) retained in `third_party/unrar/acknow.txt`.
- C/C++ boundary: `backend/extract_unrar.h` (`extern "C"`) + `backend/extract_unrar.cpp`
  (shim, not in `SOURCES`). Makefile pattern rules build
  `build/unrar/{host,ps5}/%.o` with
  `-O2 -std=c++11 -D_FILE_OFFSET_BITS=64 -D_LARGEFILE_SOURCE -DRAR_SMP -DRARDLL`
  plus `-Wno-logical-op-parentheses -Wno-switch -Wno-dangling-else` and
  `-D'__builtin_cpu_supports(x)=0'` — the PS5 link has no libgcc `__cpu_model`.
- Payload links `-lc++ -lc++abi -lunwind`; the host target uses `HOST_CXXLIBS ?= -lstdc++`.
  `Dockerfile`/`build-deps.sh` install the matching C++ toolchain.
- Make does not track flag changes: delete `build/unrar` before rebuilding after any
  `UNRAR_CXXFLAGS` edit.
- Both `payload` (`orbit_store.elf` + `orbit_runtime.elf`) and `host` targets link the 48 unrar
  objects; full Docker builds are green.

## 7. Source-aware copy (this pass)

The original implementation left Vikingfile-named copy in place; fileditch sessions would have
been announced as "Press Download on Vikingfile". Fixed everywhere:

- **Web**: `SourceId` gained `"fileditch"`; `BrowserSession` carries `source` (new
  `browser_status()` field); `BrowserNotice` heading, `Details.tsx` start-here steps and
  `Sources.tsx` intro are provider-aware ("Select any combination").
- **TV app**: `game_page.cpp` heading/steps derive the provider from `release.source_id`;
  `settings.cpp` counts fileditch as browser-only, widens the intro and the fine print;
  `fixture.cpp` now models all three providers.

## 8. Validation evidence

- `check-syntax.sh`: **0 failures** in DESKTOP and TEST modes across all backend C files, plus
  the C++ shim.
- `npm run typecheck`: exit 0. Prettier: changed files clean.
- Host functional harness: full success pipeline `outcome=0`, cancel path `outcome=2`, staging
  cleaned, archive intact, DRM patch unit-tested.
- Full Docker builds: `orbit_store.elf`, `orbit_runtime.elf`, `orbit-host` (rerun after
  revision 14 embedding).
- `python3 tools/catalog.py check`: 602 games / 687 options; `publish_catalog.py` produced
  revision 14 for both public feeds; legacy `catalogue.json` stays Archive-only (147 rows).
- TV app unit tests via `app/tools/build-tests.sh` (GoogleTest, ASan/UBSan, g++ 15): **74 tests,
  73 pass** — covering the updated status table, fixture and source parsing. The single failure
  (`Art.MarksFailuresAndEvictsTheLeastRecentlyDrawn`, expected 6 / got 4) reproduces identically
  on the pristine unmodified tree: a pre-existing clang-vs-g++ divergence in the upstream artwork
  test, not a fileditch regression. clang++ is the project's intended test compiler and is
  unavailable in this environment.

## 9. Known limitations / hardware-test TODOs

- **Cloudflare on the console**: fileditch pages are proven only through desktop automation;
  the real PS5 browser flow (JS challenge, download click, capture window) needs hardware test.
- **Artwork**: 13 of the 22 new games have no Prosperopatches title page — they keep the
  dlpsgame post image (~300 px) as cover, `ambient` layout. Post `og:description` is boilerplate
  (`"<title> PS5"`), so descriptions stay null.
- ~~**Phase-1 scope**: single-volume `.rar` only. `.partN.rar` splits and `.7z` mirrors are
  excluded by the generator (the WALL-E post is the only `.7z` case seen).~~ Superseded by
  phase 2 (§11): all 59 posts now contribute — sets, `.7z` and `.zip` included.
- **No sha256**: fileditch publishes none; verification is Status-API size equality plus the
  browser-capture validation before the transfer starts.
- **URL shape changes**: all gates fail closed — if fileditch changes its page/CDN shape, the
  catalogue compile rejects the rows instead of shipping an unsafe one.

## 10. Reproducing

```bash
# discovery (Node):        node tools/fd-crawl.mjs 83 1500 fd-crawl-results.json
python3 tools/fd_catalog.py fd-crawl-results.json --commit   # append releases + games
python3 tools/artwork.py --banners --apply                   # Prosperopatches art
python3 tools/publish_catalog.py --public-repo /path/to/repo # revision bump + feeds
python3 tools/embed.py                                       # embedded catalogue
docker run --rm -v "$PWD:/work" orbit-build:0.1 payload host # binaries
```

The complete source delta is saved as `fileditch-extraction.patch` in this repository
(vendored `third_party/unrar/` and regenerated `catalog/*` inputs are excluded; the catalogue
inputs ship through the published feeds and the generator above).

---

## 11. Phase 2 — multipart volume sets, 7z/zip, catalogue passwords

Phase 2 (requested after phase 1 shipped) lifts the single-volume restriction: dlpsgame also
publishes `.partN.rar` volume sets and `.7z`/`.zip` mirrors, and the catalogue gains an explicit
archive `password` field. Catalogue revision 15: **636 games, 724 options, 59 fileditch
releases** — 32 volume sets (2–10 volumes) and 27 singles, so all 59 dlpsgame posts now
contribute exactly one release each (previously 22).

### 11.1 Catalogue schema

```jsonc
{
  "sizeBytes": 21260088114,          // TOTAL across every volume
  "filename": "[DLPSGAME.COM]-PPSA06488.part1.rar",
  "url": "<volume-1 page>",          // top-level url/filename are always volume 1
  "password": "DLPSGAME.COM",        // <= 63 chars; empty/absent -> the unrar default
  "parts": [                         // extra volumes only (absent for single-file dumps)
    { "url": "…", "browserUrl": "…", "filename": "…part2.rar", "sizeBytes": 10630044057 }
  ]
}
```

- Evidence gate: `verification.sizeBytes == sizeBytes − sum(parts[].sizeBytes)` — the Status API
  proves volume 1, and the arithmetic binds every extra volume's declared size to it.
- `parts` exist only for `sourceId: fileditch` + `format: "Folder"`; the set must start at
  `.part1.rar` and continue contiguously with the same base name and digit padding as volume 1,
  at most 60 volumes (`FILEDITCH_MAX_PARTS` == `ORBIT_MAX_PARTS`), and every URL in the
  catalogue must be unique.
- Name gate widened in `catalog.py` and shared with the generator: `FILEDITCH_NAME` accepts the
  `[DLPSGAME.COM]-` and `[DLPSGAME.COM]_` separators (the underscore variant is real — Poppy
  Playtime, Silent Hill) plus the bare `PPSAxxxxx…` prefix (WALL-E's prefix-less `.7z`);
  `FILEDITCH_PART` parses `.partN.rar`. Suffix allowlist: `.rar .7z .zip .tar .tar.gz .tgz`.
  `fileditch_mirror_name(filename, tid)` must match exactly one title id.

### 11.2 Generator: one dump per post

`fd_catalog.py`'s `candidates()` groups a post's declared `fileditch` URL list (field order) by
`.partN.rar` base+padding and `dump_choice()` picks, in declared order: the **first complete
volume set** (≥2 members, contiguous from part 1, every declared member alive — a dead member
invalidates its whole set), else the **first single archive** passing the name gate. Sets win
over singles because the multi-volume mirror is the fuller dump. Duplicate title ids across posts
are dropped (`title id taken`), and a post whose declared `titleId` disagrees with its filenames
aborts the run.

Every chosen volume is then proven through its own fileditch page Status API (1.2 s apart) —
metadata sizes are authoritative, a moved size only warns — and one release per dump is emitted
with `parts`, `password` and volume-1 evidence (`fileditch_metadata_evidence`). All-or-nothing:
one failing kept post aborts before any write. Result of the full 59-post crawl: 37 new releases
+ 34 new games (32 sets + 5 singles; 22 earlier singles were already listed).

### 11.3 Ownership and state

- `Release.parts` is heap-owned by the catalogue; `catalog.c` frees the outgoing rows' parts
  before `apply()`. Jobs deep-copy parts at create time; state snapshots parse their own copies.
- `job_action` remove / history-clear move the parts into a `doomed` list that is freed only
  after the state file saves successfully, so a failed save can never leave a dangling pointer.

### 11.4 Download, capture, extraction

- `run_job` orchestrates volumes; per-volume `run_volume` downloads and validates. Multipart
  volumes are written straight to their **final** names (single-file jobs keep the `.part`
  scratch name), so a resume can enumerate completed volumes on disk; an identity/cursor block
  refuses progress writes when `j->part_index != k`, volumes below the cursor fast-skip, and
  multipart errors are prefixed `Part k of N: `. `job.total` is the whole-set `sizeBytes`, so
  progress spans every volume (`received` = base + current span).
- Extract runs only when every volume is present; on `ORBIT_EXTRACT_OK` all volumes are unlinked
  (the set is consumed together), otherwise the current partial is renamed to final so nothing
  re-downloads.
- Browser capture grabs every volume's page URL up front (one session, N pages, capturing
  url/etag/modified each) before a single `job_create_captured_locked`; `selected_locked`
  matches candidates by browser_url/filename/size. A signed route expiring across a long parts
  loop is recovered by the normal re-verify path.
- `extract.c` dispatch: `.rar` → vendored UnRAR with `job->release.password` (falling back to
  `DLPSGAME.COM` via `UCM_NEEDPASSWORD`); everything else → **libarchive**
  (`archive_read_add_passphrase`), covering 7z/zip/tar including AES-encrypted members.
  Extraction itself stages in `<root>/.orbit-staging-<jobid>`, BFS-locates `sce_sys/` +
  `eboot.bin`, renames into place, then hands off to the DRM patch — unchanged from phase 1.

### 11.5 Build: zlib + xz + libarchive

- `tools/build-deps.sh` fetches zlib 1.3.2, xz 5.8.1 and libarchive 3.8.5 with pinned sha256 and
  builds them for the PS5 (`--host=x86_64-pc-freebsd`, static, installed into the SDK prefix).
  libarchive is configured `--with-zlib --with-lzma --with-openssl` — 7z/zip AES needs a crypto
  backend and OpenSSL is already built in the same prefix — with every optional format/library
  disabled (`bz2, libb2, lz4, zstd, lzo2, cng, mbedtls, nettle, xml2, expat, xattr, acl`) and
  its own tools off (`bsdtar, bsdcat, bsdcpio, bsdunzip`).
- The xz build disables **all** of its CLI tools (`--disable-xz --disable-xzdec
  --disable-lzmadec --disable-lzmainfo`): PS5 clang's FreeBSD triple makes configure's link test
  for `cap_rights_limit` succeed, so the tools' Capsicum sandbox includes `<sys/capsicum.h>` —
  which the PS5 sysroot does not ship. Only liblzma (capsicum-free) is needed.
- `Makefile` links `-larchive -llzma -lz` on the payload (after `-lssl -lcrypto`) and on the
  host/test/benchmark targets; `Dockerfile` adds `libarchive-dev` for the Linux host target.
- Licences: `licenses/zlib.txt`, `licenses/GPL-2.0.txt` (liblzma is GPL-2.0-or-later — compatible
  with Orbit's GPL-3.0-or-later), `licenses/libarchive-BSD-2-Clause.txt`, three payload rows in
  `THIRD-PARTY-NOTICES.md`, inventory lines in `SOURCE-BUNDLE.md`, and `tools/release-sources.json`
  entries carrying the same pinned hashes.

### 11.6 Feed and interfaces

- `api.c` strips `password` and `parts` from the public catalogue feed (the archive password
  never leaves the console) and derives `partCount` (= parts size + 1) for the UIs. Full release
  JSON only exists in private/state job payloads.
- Job JSON gains `partCount` + `partIndex` (0-based).
- Web: `types.ts` (`Release.partCount`, `Job.partCount|partIndex`, `format` union gains
  `"Folder"`); `Details.tsx` shows an `N parts` badge with a matching aria-label; `Downloads.tsx`
  renders `Part k of N · ` in the meta line while status ∈ queued/downloading/paused/retrying —
  completed/error rows carry no label (the backend error string already has the `Part k of N:`
  prefix).
- TV: `model.hpp` (`part_count`, `Job.part_index`) + `model.cpp` parsing; `game_page.cpp` option
  detail `· N parts` and an `Archive volumes` fact row; `downloads.cpp` the same `Part k of N`
  meta line for active statuses.

### 11.7 Phase-2 validation

- `catalog.py verify --only <all 59 fileditch ids>` green (filename gate + Status API + evidence
  per release), then `build` + `check`: **636 games / 724 options**; `publish_catalog.py
  --public-repo` → **revision 15** for both public feeds; `embed.py` → `catalog.h` at revision 15
  (embedded JSON contains the multipart rows).
- `check-syntax.sh`: 0/0/0 (DESKTOP, TEST, `extract_unrar.cpp`). `tsc -b`: exit 0. Prettier:
  every changed file clean.
- Docker image rebuilt from the changed `Dockerfile`/`build-deps.sh` (zlib, xz, libarchive built
  for the PS5 toolchain), then `payload host`: `orbit_store.elf`, `orbit_runtime.elf` and
  `orbit-host` all link clean with the new libraries under `-Wall -Wextra -Werror`.
- TV tests (g++ + ASan/UBSan, host webp headers staged from Ubuntu debs): **74 tests, 73 pass** —
  identical to the phase-1 baseline; the single failure is the pre-existing
  `Art.MarksFailuresAndEvictsTheLeastRecentlyDrawn` artwork-test divergence (the environment has
  no `clang++`, so g++ stands in with `-Wno-format-truncation` for its one extra warning class).

### 11.8 Phase-2 limitations

- **Tail-volume completeness**: the crawl records only live files, so a *last* volume that died
  without leaving a gap cannot be distinguished from a complete set. A dead member that leaves a
  gap does invalidate the set; fileditch listed every declared URL alive in this crawl, and a
  set's total tracks its companion `.7z` twin within ~7% (compression variance) as a sanity
  signal — not a gate.
- **Signed-route expiry**: a long parts loop can outlive signed CDN routes on early volumes;
  downloads re-verify the route when that happens (accepted trade-off for the simple
  capture-all-upfront model).
- **Hardware test** still pending for the console browser flow and for a real multipart
  extraction on device (as in §9).
