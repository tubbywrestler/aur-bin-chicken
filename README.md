# aur-bin-chicken

A reusable GitHub Actions workflow that turns any AUR package into a precompiled
"-bin" AUR package, automatically, on a schedule.

## What it does

Compiling something big from source (an FEM solver, a compiler, anything with a
non-trivial build) is slow for every single person who installs it from the AUR.
This repo holds one reusable workflow that:

1. Watches an existing AUR package (the **base** package) for new versions.
2. When it sees a new `pkgver`/`pkgrel`, builds it from source in CI, using that
   package's own `PKGBUILD` completely untouched.
3. Publishes the compiled `.pkg.tar.zst` as a GitHub Release.
4. Automatically bumps a second, lightweight AUR package (the **-bin** package)
   whose `PKGBUILD` just downloads and unpacks that release — no compiler, no build
   time, for anyone who installs it.

You still have to push the `-bin` package's `PKGBUILD` to AUR yourself (deliberately
— see [Publishing to AUR](#publishing-to-aur) below), but everything up to that
point is fully automatic.

**Terminology used throughout:**
- **base package** — the real, existing AUR package that builds from source. Might
  be yours or a third party's; this workflow only ever reads it, never writes to it.
- **-bin package / -bin repo** — a new GitHub repo *and* a new AUR package, named
  `<base>-bin`, that this workflow creates and maintains. It hosts the CI workflow,
  the GitHub Releases (compiled binaries), and the `-bin` PKGBUILD itself.

Concrete examples in this account: `can-utils` → `can-utils-bin`,
`elmerfem-base` → `elmerfem-base-bin`, `gama` → `gama-bin`.

## How it works

`.github/workflows/sync-and-build.yml` is a `workflow_call` reusable workflow with
two jobs:

### Job `check` (`ubuntu-latest`, no container — cheap, runs every scheduled tick)

1. Clones the base package's real AUR git repo
   (`https://aur.archlinux.org/<aur_pkgbase>.git`).
2. `grep`s its `PKGBUILD` for `pkgver=`/`pkgrel=` and computes `tag = pkgver-pkgrel`.
3. Runs `gh release view <tag>` against the *calling* `-bin` repo. If that release
   already exists, this version has already been built (`is_new=false`) and the
   `build` job is skipped entirely. This is what makes the whole pipeline stateless
   and idempotent — the `-bin` repo's own list of GitHub Releases **is** the record
   of "have we built this version," no separate database or tracking file needed.

### Job `build` (only runs when `is_new == 'true'`, `container: archlinux:base-devel`)

1. Bootstraps `git` and `github-cli` — the `archlinux:base-devel` image ships
   **neither** by default, so both are installed explicitly before anything else
   can happen.
2. Clones the base package fresh, sources its `PKGBUILD` to read `depends`/
   `makedepends`, and `pacman -S`s them.
3. Creates an unprivileged `builder` user (`makepkg` refuses to run as root — this
   also means any test suite that shells out to `mpirun` etc. runs as a normal user,
   as it should).
4. Builds with `makepkg` (or `makepkg --nocheck` if `run_tests: false` — see
   [Inputs](#inputs)) as `builder`. This runs the base package's own
   `prepare()`/`build()`/`check()`/`package()` completely untouched.
5. Publishes the resulting `.pkg.tar.zst` as a GitHub Release tagged `<tag>` in the
   calling `-bin` repo.
6. Checks out the `-bin` repo itself and bumps its own `PKGBUILD` in place:
   `pkgver`, `_pkgrel_src` (the *base* package's `pkgrel`, kept distinct from the
   bin package's own), `pkgrel` (reset to `1`), and `sha256sums` (the real hash of
   the artifact just built — not a placeholder for `updpkgsums` to fill in later).
   Regenerates `.SRCINFO`, commits, and pushes to `master`.
7. Updates a second **`aur` branch** containing only `PKGBUILD`, `.SRCINFO`, and
   `.gitignore` — no `.github/` — built as a normal incremental commit on top of
   whatever that branch already contains (an orphan commit only the very first
   time it's created), and pushes it. This is the branch you actually push to AUR
   (see below for why, and why it must never be regenerated from scratch after
   the first run).

A job in a called reusable workflow runs with the *caller's* `github.repository`/
`GITHUB_TOKEN` context, not `aur-bin-chicken`'s — that's what makes `gh release
create`/`view` and the `git push` in step 6/7 correctly target whichever `-bin` repo
invoked the workflow, with zero extra plumbing.

## Inputs

| Input | Required | Default | Meaning |
|---|---|---|---|
| `aur_pkgbase` | yes | — | The AUR `pkgbase` to track and build, e.g. `can-utils`, `elmerfem-base`, `gama`. |
| `run_tests` | no | `true` | Whether to let the base package's own `check()` run. Set `false` to build with `makepkg --nocheck` — useful for slow/flaky suites (MPI-heavy ones especially) where you're trusting upstream's own testing rather than re-running it on every CI build. |

## Setting up a new `<foo>-bin` package

Say `foo` is an existing AUR package (yours or someone else's) and you want
`foo-bin`:

1. **Create the GitHub repo**: `gh repo create <you>/foo-bin --public`.
2. **Clone it locally** and add `.github/workflows/sync.yml`:
   ```yaml
   name: Sync foo

   on:
     schedule:
       - cron: '17 4 * * *'
     workflow_dispatch:

   jobs:
     sync:
       uses: <you>/aur-bin-chicken/.github/workflows/sync-and-build.yml@master
       with:
         aur_pkgbase: foo
         # run_tests: false   # uncomment to skip check()
       permissions:
         contents: write
   ```
3. **Add the `-bin` PKGBUILD template.** Keep `pkgver`, `_pkgrel_src`, `pkgrel`, and
   `sha256sums` as plain, single-line assignments exactly like this — the workflow's
   `sed` commands rewrite them in place on every build:
   ```bash
   # Maintainer: you

   pkgname=foo-bin
   pkgver=<foo's current pkgver>
   _pkgrel_src=<foo's current pkgrel>
   pkgrel=1
   pkgdesc="... (precompiled)"
   arch=('x86_64')
   url="..."
   license=(...)
   options=('!debug')   # nothing to strip - the binaries are already built
   provides=('foo')
   conflicts=('foo')
   depends=(...)   # foo's runtime depends only - no makedepends, nothing compiles

   source=("https://github.com/<you>/foo-bin/releases/download/${pkgver}-${_pkgrel_src}/foo-${pkgver}-${_pkgrel_src}-x86_64.pkg.tar.zst")
   sha256sums=('SKIP')   # CI overwrites this with the real hash on first build

   package() {
       bsdtar -xf "${srcdir}/foo-${pkgver}-${_pkgrel_src}-x86_64.pkg.tar.zst" -C "${pkgdir}" --exclude .PKGINFO --exclude .BUILDINFO --exclude .MTREE
   }
   ```
4. Generate `.SRCINFO` (`makepkg --printsrcinfo > .SRCINFO`), add a `.gitignore`
   (`/pkg`, `/src`, `*.pkg.tar.*`), commit, push to `master`.
5. Trigger the first build manually rather than waiting for the schedule:
   `gh workflow run sync.yml --repo <you>/foo-bin`.
6. Once green, `gh release list --repo <you>/foo-bin` shows the new tag, and an
   `aur` branch now exists on the repo containing just the three flat files.

## Publishing to AUR

Read these first:
- [Arch Wiki: Arch User Repository](https://wiki.archlinux.org/title/Arch_User_Repository)
  — account/SSH prerequisites, how AUR's git hosting works, submitting a new package.
- [Arch Wiki: AUR submission guidelines](https://wiki.archlinux.org/title/AUR_submission_guidelines)
  — naming, licensing, and quality rules AUR expects before you submit.

### One-time account setup

1. Register an AUR account at [aur.archlinux.org/register](https://aur.archlinux.org/register)
   if you don't have one.
2. Generate a dedicated SSH keypair for this (or reuse an existing one), and add the
   **public** key to your account under *My Account → SSH Public Keys* on
   aur.archlinux.org.

### Why not just push `master`

**AUR's git hooks reject any push whose tree contains a subdirectory** — not a
filename whitelist, just a flat-files-only rule. `master` has `.github/workflows/`
on it (required for GitHub Actions to even discover the workflow), so it can never
be pushed to AUR directly. The `aur` branch the workflow maintains (see
[How it works](#how-it-works), step 7) exists specifically to solve this: it only
ever contains the flat files AUR needs.

**Important:** that branch is built as a normal incremental commit on top of
whatever it already contains, never regenerated as a fresh orphan after its first
commit. AUR's server *also* rejects non-fast-forward pushes to `master` — it will
not accept rewritten history, even via `--force`. An earlier version of this
workflow regenerated the `aur` branch from scratch on every run, which seemed
simpler and safer (no merge conflicts possible) since AUR doesn't care about git
history — but it produces a branch with no shared ancestry between runs, so every
push after the first one gets rejected server-side with `denying non-fast-forward
... hook declined to update refs/heads/master`. Learned this the hard way; don't
reintroduce it.

### One-time per-package setup

```bash
cd foo-bin
git remote add aur ssh://aur@aur.archlinux.org/foo-bin.git
```

### Syncing down

CI owns the `aur` branch — never hand-edit it. Before every push, refresh your
local copy from what CI most recently generated on GitHub:

```bash
git fetch origin aur:aur -f
```

### Pushing up

```bash
git push aur aur:master
```

**First push** (standing up the package for the first time): AUR's repo is empty,
so this is trivially a fast-forward. This single push *is* the act of creating/
registering the package on AUR; there's no separate "submit" button or step.

**Every push after that**: since the `aur` branch now has real, connected history
(each build's commit descends from the last), a plain push is *still* a
fast-forward — no `--force` ever needed, and none should be used. If you ever see
`denying non-fast-forward` / `hook declined to update refs/heads/master` here, it
means your local `aur` branch has drifted from what's actually on AUR (e.g. after
manually editing the repo, or if a previous version of this workflow's bug
regenerated it as an orphan) — resolve that by rebuilding `aur` on top of AUR's
actual current tip (`git fetch aur master:aur-real`, reapply
`PKGBUILD`/`.SRCINFO`/`.gitignore` from `master` on top, commit, then force-update
just the *local/GitHub* copy of `aur`, never AUR itself), rather than reaching for
`--force` on the push to AUR.

Do **not** `git push -u`/`--set-upstream` this branch to the `aur` remote. Its real
source of truth is GitHub (`origin/aur`, written by CI) — AUR is a one-way publish
target you push to on purpose, never something you want a bare `git pull` to
accidentally point at.

**Note:** after pushing, AUR's web search/RPC index can lag behind the git push by
some time, even though the git repo and the package's own page
(`aur.archlinux.org/packages/<name>`) update immediately. Don't be alarmed if
`yay -Sy <name>` or the search box doesn't find it right away — retry later rather
than assuming the push failed.

## Troubleshooting / known gotchas

These were all discovered the hard way while building this pipeline — noted here so
nobody has to rediscover them:

- **AUR rejects non-fast-forward pushes to `master` — never regenerate the `aur`
  branch as a fresh orphan after its first commit.** It's tempting (and was our
  first implementation): AUR doesn't care about git history, so why not just wipe
  and recreate the branch every run? Because AUR's server independently enforces
  fast-forward-only updates regardless of history *content* — an orphan commit has
  no shared ancestry with the previous one, so the push is rejected with `denying
  non-fast-forward` / `hook declined to update refs/heads/master`, even with
  `--force` on the client side (that only overrides local safety checks, not the
  server's). The `aur` branch must be built incrementally: fetch it if it exists,
  commit the current `PKGBUILD`/`.SRCINFO`/`.gitignore` on top, push normally.
- **VCS packages with a dynamic `pkgver()` function** (version computed via
  `git describe` etc. at build time, e.g. `cangaroo`) can cause `makepkg` to rewrite
  the checked-out `PKGBUILD`'s `pkgver` in place if it computes something newer
  than what's checked in on AUR. The workflow re-reads `pkgver`/`pkgrel` from
  `aur-src/PKGBUILD` *after* the build (`Read actual built version` step) rather
  than trusting the `check` job's pre-build values, so the release tag, the
  uploaded artifact's filename, and the `-bin` PKGBUILD's `source=` URL always
  agree — for ordinary static-`pkgver` packages this just reproduces the same
  value the `check` job already found, so it's a no-op there.
- **`archlinux:base-devel` ships neither `git` nor `gh`.** Both are installed
  explicitly, up front, before anything else — they can't wait for "install the base
  package's own `makedepends`" since that's not guaranteed to include either.
- **Container jobs default to `sh`, not `bash`.** In this image `/bin/sh` is bash
  running in POSIX-compatibility mode, which changes behavior in two ways that bit
  us: `source <file>` does a `$PATH` search instead of checking the current
  directory (so `source PKGBUILD` fails with `file not found` even though the file
  is right there), and array syntax (`"${arr[@]}"`) doesn't work at all. Fixed with
  `defaults: run: shell: bash` on the `build` job.
- **Never `chown` (even non-recursively) the `-bin` repo's checkout** when giving
  the unprivileged `builder` user write access for `makepkg --printsrcinfo`. Modern
  git treats a worktree whose top-level directory is owned by someone other than the
  current user as "dubious ownership" and refuses plain commands like `git config`/
  `git commit` in the very next (root) step, failing with the confusingly-unrelated
  message `fatal: not in a git directory`. Use `chmod o+w` instead — `makepkg` only
  needs write *permission*, not ownership.
- **`makepkg` refuses to run as root, even for `--printsrcinfo`.** There's no flag
  to override this for metadata-only use; the unprivileged `builder` user is
  required for every `makepkg` invocation, not just full builds.
- **Always set `options=('!debug')` in the `-bin` PKGBUILD.** Without it, `makepkg`
  tries to auto-generate a companion `-debug` package by extracting debug symbols
  from binaries that are already stripped (they came pre-built from the base
  package) — this fails on every file with harmless-looking but noisy
  `gdb-add-index: No debugging symbols` errors. Separately, the *base* package's own
  build can also produce a `*-debug-*.pkg.tar.zst` if its own `PKGBUILD` doesn't set
  this — the workflow's release/hashing steps explicitly filter those out, since the
  `-bin` PKGBUILD's `source=` only ever references the non-debug artifact.

## Files each `-bin` repo needs

- `PKGBUILD` — see the template above; CI rewrites four fields in place on every run.
- `.SRCINFO` — kept in sync by CI, but must exist (and be regeneratable via
  `makepkg --printsrcinfo`) from the start.
- `.gitignore` — recommended: `/pkg`, `/src`, `*.pkg.tar.*`.
- `.github/workflows/sync.yml` — the short caller workflow (see above).
