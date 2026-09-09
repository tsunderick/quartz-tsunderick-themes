# quartz-tsunderick-themes

The **forkable base** for every Tsunderick/Ashfall site: the Quartz v5 core, the
house design system (theme switcher with 17 themes, font switcher with
self-hosted Operator Mono + JetBrains Mono), and the house plugins
(`toc-true-depth`, `folder-alpha`, `h1-title`, `graph-labels`,
`backlinks-collapse`, `reader-zen`).

Each site is a **self-contained fork of this repo**: it carries its own
content + identity, builds with plain npm, and pulls house improvements by
merging from `base`. This repo is the product — sites fork _this_ code, not
upstream.

Based on [jackyzha0/quartz](https://github.com/jackyzha0/quartz); the upstream
remote is configured and upstream merges remain possible but non-driving (see
the appendix).

## The model in one minute

```
 you ──▶ quartz-tsunderick-themes (base: themes, plugins, quartz core)
              │
              │   git fetch base && git merge base/main  (= quartz-sync sync)
              ├──▶ 7th-heaven.tsunderick.space ─ push ─▶ Cloudflare deploy
              ├──▶ docs.ashfallsoftware.com     ─ push ─▶ Cloudflare deploy
              └──▶ family.tsunderick.space/docs ─ push ─▶ deploy (subtree)
```

The fleet today:

| Site                           | Flavor     | Sync                          |
| ------------------------------ | ---------- | ----------------------------- |
| `7th-heaven.tsunderick.space`  | flat       | `quartz-sync sync`            |
| `docs.ashfallsoftware.com`     | flat       | `quartz-sync sync`            |
| `family.tsunderick.space/docs` | subtree    | `sync --prefix docs`          |

- **Forks own everything they show**: `content/`, `quartz.config.yaml` identity
  (pageTitle, baseUrl, colors, fonts, plugin toggles), `README.md`, homepage,
  `quartz/static/` branding.
- **The base owns what every site should inherit**: quartz core, house plugins,
  house styles, `package.json`, CI, `.gitattributes` sync policies.
- **Push = deploy**: a fork's push to `main` triggers its Cloudflare build.

## The invariant: sync ≠ build

Rust exists only on the **sync** path (`tools/quartz-sync`). Building a site
never needs it:

```
npm ci && npx quartz plugin install && npx quartz build
```

must work on a machine with zero Rust installed. Cloudflare CI never invokes
cargo. A fork that never syncs never compiles anything.

## Syncing base improvements into a site

### The easy way — `quartz-sync`

```
cd tools/quartz-sync && cargo build --release   # once, on your dev machine
../..                                          # repo root
tools/quartz-sync/target/release/quartz-sync setup  # once, idempotent
tools/quartz-sync/target/release/quartz-sync sync   # fetch/merge/resolve/commit
```

`sync` (default ref `base/main`):

1. fetches `base` and merges `--no-commit --no-ff`,
2. resolves a conflicted `quartz.config.yaml` **semantically** — per-key 3-way:
   your intentional changes win, base's changes flow where you didn't touch,
   both-changed → yours + a warning naming the path,
3. **fork-identity guard**: keeps your `README.md` and `content/index.md` even
   when base alone changed them (`merge=ours` attributes only fire on conflicts
   — one-sided changes would otherwise flow in silently),
4. runs `npm install`echo "static site — no build step" to reconcile the
   lockfile when `package.json` changed (set `QUARTZ_SYNC_SKIP_NPM=1` to skip),
5. auto-commits clean syncs as `sync: base <old>..<new>`; anything it can't
   resolve exits nonzero with guidance (git's conflict markers are left in
   place; `git merge --abort` rolls back).

### The manual way (no tooling required)

```bash
git fetch base
git merge base/main --no-commit
# resolve quartz.config.yaml by hand if base touched it
git restore --source=HEAD --staged --worktree -- \
  README.md content/index.md                  # only if base changed them
npm install                      # if package.json changed
git commit
```

### Monorepo sites (`--prefix`)

When the fork lives in a **subdirectory of a monorepo** (like
`family.tsunderick.space`'s `docs/`), the base tree is subtree-mapped into that
directory and everything else in the monorepo is structurally untouchable by
base:

```bash
quartz-sync sync --prefix docs      # run from the MONOREPO ROOT
```

Same flow as the flat `sync`, but the merge runs with `-Xsubtree=docs`, the
semantic config resolution and identity guard operate on
`docs/quartz.config.yaml` / `docs/content/index.md` / `docs/README.md`,
`npm install` runs inside `docs/`, and base's files can only ever land under
`docs/`. Manual equivalent:

```bash
git fetch base
git merge -Xsubtree=docs --no-commit --no-ff base/main
# resolve docs/quartz.config.yaml by hand if base touched it
git restore --source=HEAD --staged --worktree -- \
  docs/README.md docs/content/index.md         # if base changed them
(cd docs && npm install)           # if package.json changed
git commit
```

Monorepo subpath sites also carry a fork-owned build script (e.g.
`docs/scripts/build.sh`) that copies the flat `public/` output into `dist/docs/`
so the worker's `/docs*` route sees the right layout — see "Creating a new site"
below.

### What auto-resolves vs. what's manual

| Path                                 | Behavior                             |
| ------------------------------------ | ------------------------------------ |
| `quartz.config.yaml`                 | semantic 3-way resolver              |
| `README.md`, `content/index.md`      | always yours (ours driver + guard)   |
| `package-lock.json`, `.node-version` | ours on conflict; base bumps flow in |
| `quartz/styles/custom.scss`          | manual only on same-region edits     |
| everything else                      | normal git merge                     |

**Absence is deletion.** Fork configs are full-ownership documents: a key
missing from your config is an intentional removal, not "inherit from base"
(that was the old engine's layering, now retired). To keep a key, leave it in
the file.

### When a sync conflicts

Things that **can never conflict**: your `README.md`, homepage, `content/`,
branding (identity guard), your plugin additions/deletions (fork wins), base's
new files (they just arrive).

**`quartz.config.yaml`** is the hotspot and it's _auto_-resolved — you'll see
warnings like:

```
⚠ $.configuration.pageTitle: both sides changed; kept the fork's (fork wins)
```

That's the resolver telling you exactly what it kept and why; no action needed.
Real conflict markers only appear for structural surprises (duplicate plugin
sources) or the **one known manual case**: same-region edits to
`quartz/styles/custom.scss` on both sides.

If a sync exits nonzero and leaves markers:

```bash
git diff --name-only --diff-filter=U     # what's unresolved
# edit the file(s) your way, then:
git add <file> && git commit
# or bail out entirely — nothing is lost:
git merge --abort
```

Already committed something bad? `git revert -m 1 HEAD` — and the previous
deployment stays live on Cloudflare until you push the fix.

## The golden rule: fix it in the base

If you find yourself editing the same thing in more than one fork, it belongs
here. Ship it in the base, then one `sync` per site. Forks stay tiny: content +
identity.

## Creating a new site

Two flavors. Identity is fork-owned from day one in both, so it can never
conflict with base.

### Flavor A — own repo, repo root (like 7th-heaven / ashfall)

```bash
git clone https://github.com/tsunderick/quartz-tsunderick-themes.git my-site
cd my-site
git remote rename origin base                      # the sync source
git remote add origin git@github.com:YOU/my-site.git
```

Then make it yours:

- `content/` — markdown, starting with `index.md` (replace base's preview
  homepage)
- `quartz.config.yaml` — edit the **identity**: `pageTitle`, `baseUrl`,
  `locale`, `theme.typography`, `theme.colors`, the theme-switcher `themes` menu
- `README.md` — yours; branding → `quartz/static/icon.png` + `og-image.png`

```bash
git add -A && git commit -m "site: claim identity"
git push -u origin main   # after creating the empty repo
```

The clone point is the merge base, so every future sync is a plain related merge
— no conversion ceremony needed. Build locally:

```bash
mise exec node@$(cat .node-version) -- npx quartz build --serve
# serves at http://localhost:8080
```

Deploy: connect the repo to a Cloudflare Workers build (root directory `/`).
Three pieces, two owners — the **dashboard** holds the commands, the **fork**
owns the config:

| Piece | Lives where | What it does |
|---|---|---|
| Build command | CF dashboard | `git fetch --unshallow && npm ci && npx quartz plugin install && npx quartz build` — compiles the site into `public/` |
| Deploy command | CF dashboard | `npx wrangler versions upload` — runs wrangler after the build |
| `wrangler.jsonc` | **fork repo root** | tells wrangler the worker name + that it serves `./public` — without it the deploy fails with `Missing entry-point to Worker script or to assets directory` |

Fork setup:

```bash
cp templates/wrangler.jsonc wrangler.jsonc   # then rename "name" to your CF project name
```

The template's comments are load-bearing: **the trailing commas are required**
(`.prettierrc` sets `trailingComma: "all"`, which prettier enforces on
`.jsonc` too — removing them fails `npm run check` / engine-ci). Run
`npm run check` before pushing; a red check on a fork is almost always this.

### Flavor B — subdirectory of a monorepo (like family.tsunderick.space/docs)

Graft the base tree into your chosen subdirectory with git's subtree mapping —
the monorepo root is then **structurally untouchable by base** forever:

```bash
cd my-monorepo
git remote add base https://github.com/tsunderick/quartz-tsunderick-themes.git
git fetch base
git merge --allow-unrelated-histories -Xsubtree=wiki base/main
```

A **fresh** subdirectory lands with zero conflicts — the whole base tree simply
appears under `wiki/`. Claim identity inside `wiki/` exactly as in Flavor A,
then register the drivers at the monorepo root:

```bash
# after: cd wiki/tools/quartz-sync && cargo build --release
wiki/tools/quartz-sync/target/release/quartz-sync setup
git add .gitattributes && git commit -m "setup: quartz-config merge driver"
```

Future syncs: `quartz-sync sync --prefix wiki` (or the manual `-Xsubtree=wiki`
recipe above).

**Serving under a path** (`domain.com/wiki`) rather than a subdomain? Quartz
derives all URLs from `baseUrl` (`domain.com/wiki`) natively, but the worker's
route serves assets by URL path — so the deployable tree must literally contain
`wiki/index.html`. Add a fork-owned build script (`wiki/scripts/build.sh`):

```bash
npx quartz build
rm -rf dist && mkdir -p dist/wiki && cp -r public/. dist/wiki/
```

…point `wiki/wrangler.jsonc` `assets.directory` at `./dist`, and append
`&& bash scripts/build.sh` to the build command. Own subdomain → skip all of
this; the plain build works.

Deploy: Cloudflare Workers build with **root directory `/wiki`** and the npm
triple build command above.

## House plugins & styles (what forks inherit)

Engine-enforced defaults beyond upstream Quartz: `toc-true-depth` (order 51 —
true heading levels in the TOC, pairs with the H-prefix styles in house
`custom.scss`), `folder-alpha`, `h1-title`, `theme-switcher`, `font-switcher`,
`graph-labels`, `backlinks-collapse` (backlinks with a fold-away header,
mirroring the TOC), `reader-zen` (reader mode with persistent _italic_ /
_full-width_ zen sub-options while reading).

**Right-sidebar canonical order** (house-enforced): TOC (priority 30) →
backlinks (50) → graph (60, bottom). The graph is the house
`./plugins/graph-labels` wrapper (always-visible node labels); upstream
`@quartz-community/graph` stays listed-but-disabled as the visible opt-anchor.
Opt out per site: disable the wrapper entry (and re-enable upstream), or change
a `priority`.

**Cascade-layering hazard (house styles vs plugin CSS):** the emitted stylesheet
is `@layer quartz-base { …core + plugin CSS… }` followed by `custom.scss`
**unlayered** — and unlayered declarations beat layered ones _regardless of
specificity_. A house rule like `.sidebar .toc { flex: 0 0 auto }` will silently
kill a plugin's layered interactive-state rule
(`.toc:has(button.toc-header.collapsed) { flex: 0 1 1.4rem }` — this exact bug
shipped once). Any style that must coexist with an interactive plugin state
(`.collapsed`, `[reader-mode=on]`, …) needs an explicit stand-down guard or
high-specificity state-scoped selectors — see the guards throughout
`quartz/styles/custom.scss`.

**H1 in the TOC:** `h1-title` splices the page's single H1 out of the body as
the article title before the TOC transformer runs — so `toc-true-depth` prepends
it back as a synthetic first entry (`H1 <title>`, linking to the shared
`#article-title` anchor). Pages with no other TOC entries keep their no-TOC
rendering. Opt out with `includeH1: false` on the `./plugins/toc-true-depth`
entry.

**Serving under a subpath:** Quartz v5 derives the page basePath from the config
`baseUrl` path (`domain.com/docs`), so links/SPA/404 honor the prefix natively.
The engine-era output re-structuring (`public/<subpath>/index.html`) retired
with `build.sh` — if a deploy target literally needs that layout, post-process
the build.

## The sync toolkit (`tools/quartz-sync`)

Single Rust crate, deliberately tiny (`serde_yaml_ng` only, isolated behind
`src/yaml_io.rs` so the YAML crate is a one-file swap):

```
src/merge.rs    pure semantic 3-way core (no I/O; 20 unit tests, incl. the
                four migration fixtures: base-adds-plugin · base-fixes-
                options+fork-identity · fork-disables-inherited · both-touch-
                theme-switcher)
src/yaml_io.rs  parse/emit boundary + leading-comment (schema directive) capture
src/driver.rs   git merge-driver entrypoint (`driver %O %A %B %P`)
src/sync.rs     fetch/merge/resolve/guard/lockfile/auto-commit orchestration
src/setup.rs    idempotent driver registration (quartz-config + ours — note:
                git ships NO built-in "ours" driver; `setup` defines it)
```

`setup` also registers the conventional `ours` merge driver
(`merge.ours.driver = true`) — without it the `.gitattributes` ours-policies are
inert for plain `git merge`.

## Upstream Quartz merges (appendix — possible, not driving)

This repo keeps the full Quartz history, so upstream updates are plain merges:

```bash
git fetch upstream
git merge upstream/v5
# resolve conflicts (quartz/ core is pristine, so this is usually clean)
npm ci && npx quartz plugin install && npm test && npm run check
git push            # forks receive it via their next sync
```

## Base-only development

The base self-hosts a theme preview (`content/index.md`):
`npx quartz build --serve` here. `npm test && npm run check` gate the
core; the toolkit gate:

```bash
cd tools/quartz-sync
cargo fmt --check && cargo clippy --all-targets -- --deny warnings
cargo test
```

## Troubleshooting

### `sharp: Attempting to build from source via node-gyp`

sharp ships prebuilt binaries (the `@img/sharp-*` optional deps). On machines
with a **system-wide libvips** (`pkg-config --modversion vips-cpp`, e.g. Arch),
sharp's install script detects it, rejects the prebuilt, and insists on
compiling — which fails without node-gyp. Workaround:

```bash
SHARP_IGNORE_GLOBAL_LIBVIPS=1 npm ci
```

(`npm ci --ignore-scripts && npm rebuild` also works — the module itself is
fine; only its check script misfires.)

### Native dep failures under the wrong node

Run the pinned version: `mise exec node@$(cat .node-version) -- npm ci`.

### `ERESOLVE could not resolve` (`rehype-typst` peer conflict) — latent

This repo pins `@myriaddreamin/rehype-typst@^0.6.0`;
`@quartz-community/latex@0.1.0` declares `rehype-typst@^0.5.0` as an optional
peer. `.npmrc` sets `legacy-peer-deps=true`, suppressing the conflict. Harmless
today — removing the flag or bumping either package will resurface it until
versions re-align.
