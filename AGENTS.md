# AGENTS.md — soulbeet fork (homelab deployment)

This repo is a **fork of terry90/soulbeet** deployed as part of a homelab stack.
It lives at `~/homelab/services/soulbeet/` inside the homelab repo. The soulbeet
service is built from this fork and deployed via `docker-compose.yml` in this
directory.

## Fork maintenance: staying in sync with upstream

This is a fork, not a clone of the original. Two remotes are configured:

| Remote | URL | Purpose |
|---|---|---|
| `origin` | `git@github.com:samjohnson729/soulbeet.git` | Your fork — push your changes here |
| `upstream` | `https://github.com/terry90/soulbeet.git` | The original project — fetch updates from here |

### Getting upstream updates

`soulbeet` is actively developed by terry90. To bring in upstream changes:

```bash
git fetch upstream
git rebase upstream/master
git push origin master
```

- `git fetch upstream` downloads terry90's latest commits without touching your
  working tree.
- `git rebase upstream/master` replays your local commits on top of upstream's
  latest master. Resolve any conflicts, then continue the rebase.
- `git push origin master` pushes your rebased branch to your fork.

**Note:** `git pull` with no arguments only pulls from `origin` (your fork). It
will **not** pull from terry90/soulbeet. Always use `git fetch upstream` +
`git rebase upstream/master` explicitly.

### Working on changes

For a personal homelab fork, committing directly to `master` is fine. If you
accumulate several independent changes (a bug fix, a capability tweak, a config
change), create a feature branch so they stay separable:

```bash
git checkout -b fix/some-bug
# make changes, test, commit
git checkout master
git merge fix/some-bug
git push origin master
```

## Building and deploying the image

The `docker-compose.yml` builds the image from this fork using Docker Compose's
`build:` directive, with the `TIER` build arg selecting the tier:

```yaml
build:
    context: .
    args:
        - TIER=full
```

Tiers: `light` (musicbrainz only), `medium` (adds chroma/AcoustID), `full`
(adds cover art, lyrics, ReplayGain, and more). `:light` is the default.

To rebuild and redeploy:

```bash
cd ~/homelab
python3 homelab.py up soulbeet --build
```

The build compiles Rust + Node + beets, so it takes several minutes on first
build. Subsequent builds are cached and much faster.

## Configuration files in this repo

| File | Purpose |
|---|---|
| `docker-compose.yml` | Service definition. Mounts host paths for data, downloads, music, beets config. Uses `BEETSDIR=/config` so the mounted beets config replaces image defaults entirely. |
| `.env.example` | Template for `.env`. Only `SOULBEET_SECRET_KEY` is needed here. |
| `.env` | **Gitignored.** Contains `SOULBEET_SECRET_KEY`. Copy from `.env.example` and generate with `openssl rand -hex 32`. |
| `beets/config.yaml` | Custom beets config mounted at `/config`. Edited on the host, picked up on restart. `search_limit: 20`, `extra_tags` set here. |

## Host paths (from docker-compose.yml)

| Host path | Container target | Contents |
|---|---|---|
| `/blueshades/services/soulbeet/data` | `/data` | Soulbeet SQLite DB (users, settings, slskd URL + API key, scrobble tokens) |
| `/blueshades/staging/slskd/downloads` | `/downloads` | slskd's completed transfers (shared with slskd and conductor) |
| `/blueshades/media/music/library/soulbeet` | `/music` | Imported library, plus per-folder `.beets_library.db` |
| `services/soulbeet/beets` | `/config` | Beets config directory (read-only bind mount) |

## Relationship to the homelab stack

- **Conductor** is stopped while soulbeet runs. Both read slskd's completed
  transfers from the same staging directory; running both simultaneously risks
  one moving files out from under the other (see the retired standalone beets
  service history in `services/music/README.md`).
- **slskd-routing** is unaffected: `route_music = false`, so it never touches
  music.
- **Navidrome** mounts `/blueshades/media/music/library/soulbeet` read-only and
  scans it. It is configured with `ArtistImageFolder=/music/soulbeet/artist_art`.
- **Caddy** proxies `soulbeet.blueshades.me` with `private_auth` (RFC1918 +
  Tailscale only). The container publishes only to `127.0.0.1:9765`.

## Local development server

You do **not** need to rebuild the Docker image to test code changes. soulbeet
is a Dioxus/Rust web app with a local dev server that supports hot reload:

```bash
# 1. Install prerequisites (one-time)
#    Rust/Cargo: https://rustup.rs/
#    dioxus CLI: cargo install dioxus-cli@0.7.4 --locked
#    Node.js (v22) for Tailwind: used by ./css.sh

# 2. Run the Tailwind CSS watcher (rebuilds CSS on change)
./css.sh

# 3. In another terminal, run the dev server
dx serve --platform web
```

The dev server rebuilds incrementally as you edit Rust/frontend code, so it's
much faster than the multi-stage Docker build (which compiles Rust, bundles
Node/Tailwind, and builds beets).

### Running tests

The Python tests need Python 3.11+ with beets and pytest installed:

```bash
cargo test --workspace --features server,dioxus/server --lib
cargo test -p soulbeet --lib
python -m pytest -q lib/soulbeet/tests/probe lib/soulbeet/tests/import
```

## Important notes

- The image's `beets_config.yaml` is **templated at build time** from
  `build/tiers/${TIER}.env`. The committed `beets/config.yaml` in this repo is
  mounted over `/config` and replaces the image's shipped config entirely via
  `BEETSDIR=/config`. Do not use `BEETS_CONFIG` — it layers on top of image
  defaults and can silently clobber sections like `musicbrainz:`.
- The previous deployment used a local `debug-soulbeet:latest` image with a
  filename fallback fix in `monitor.rs`. That fix should be evaluated as part of
  the first bugfix branch against upstream: check whether it has landed in
  terry90/soulbeet, and if not, carry it as a local commit or submit it upstream.
- `beets/library.db` is a runtime artifact and is gitignored. It should not be
  committed.

## Useful commands

```bash
# Check status
python3 homelab.py status
python3 homelab.py logs soulbeet    # follows logs

# Rebuild and restart
python3 homelab.py up soulbeet --build

# Stop
python3 homelab.py down soulbeet

# Run beets inside the container (debugging)
docker exec -it soulbeet-soulbeet-1 beet --version

# Check upstream for updates
git fetch upstream
git log ..upstream/master --oneline   # see what's new upstream

# Local development (no Docker rebuild needed)
./css.sh              # Tailwind watcher, run in one terminal
dx serve --platform web  # Dev server with hot reload, run in another

# Tests
cargo test --workspace --features server,dioxus/server --lib
cargo test -p soulbeet --lib
python -m pytest -q lib/soulbeet/tests/probe lib/soulbeet/tests/import
```
