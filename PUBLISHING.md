# Publishing azubinachweis to Typst Universe

Everything below is automated by [`publish.sh`](publish.sh), included at the
end of this file. It runs on Arch Linux with `pacman`, `git` and `gh`.

GitHub account: [`AyloRyd`](https://github.com/AyloRyd)

---

## In short

```sh
sudo pacman -S --needed typst git github-cli oxipng ttf-liberation
gh auth login

./publish.sh --check      # validate only, nothing remote is touched
./publish.sh --dry-run    # build, copy, commit locally — no push, no PR
./publish.sh              # the whole way, asks once before pushing
```

Work up through the three modes. Each prints `PASS` or `FAIL` per check and
stops at the first real problem, so a clean `--check` means the package is
submittable and a clean `--dry-run` means the commit is ready to review.

Only `--check` is genuinely offline. The first `--dry-run` forks
`typst/packages` to your GitHub account (step 7) — it has to, before it can
clone it — so the very first one does leave a trace remotely. It still pushes
nothing and opens nothing.

---

## What the script does

| Step | What happens | Fails when |
| --- | --- | --- |
| 1 Locate the package | finds the directory holding `typst.toml` — next to the script, or under `<name>/<version>/` — and checks the manifest is the one the script targets | no `typst.toml` is found nearby, or it names a different package/version |
| 2 Tools | `typst`, `git`, `gh`, `oxipng`, `python3` present; typst new enough for `compiler`; a sans-serif from the font stack installed; `gh` logged in | a tool is missing, typst is too old, or gh is not authenticated |
| 3 Package contents | required files; manifest `name`/`version` match the folder; required manifest fields; `[template]` present; template imports `@preview/…` and not `../lib.typ`; every version reference current; both READMEs free of dead links; LICENSE matches the manifest | any of it is off |
| 4 Build | the package is staged into `$XDG_DATA_HOME/typst/packages/preview` so the examples can resolve their own `@preview/…` import, then every example compiles and its preview image is rendered | an example fails to compile or spills onto a second page |
| 5 Install and test | the staged copy is reinstalled, now carrying the freshly built examples and images, then `typst init` into a temp dir and compiled | `typst init` fails or the fresh template is not exactly one page |
| 6 Thumbnail | rendered from the *initialised* template at 250 PPI, optimised, measured | long edge < 1080 px, file > 3 MiB, or it is referenced inside the package |
| 7 Fork and clone | forks `typst/packages` and clones your fork sparsely, or reuses an existing checkout | the fork cannot be cloned |
| 8 Copy | recreates the branch `azubinachweis-0.1.0` from a freshly fetched `upstream/main`, then copies the package into `packages/preview/azubinachweis/0.1.0` | the version is already on `upstream/main`, or the checkout has unrelated changes |
| 9 Commit | stages and commits — always exactly one commit on top of `upstream/main`, however often the script is run | — |
| 10 Push and PR | pushes and opens the pull request | you decline the prompt |

Steps 1–6 run in every mode. `--check` stops after 6, `--dry-run` after 9.

### Two details worth knowing

**It can be run from either place.** The script finds the package itself, so it
works whether it sits next to `typst.toml` — as it does in this repository —
or at the top of an archive next to `<name>/<version>/`. Override the search
with `SRC_DIR=/path/to/the/package`.

**The checkout is scratch space, not a place to edit.** Step 8 rebuilds the
branch from `upstream/main` and copies the package in from here, so anything
changed by hand under `~/projects/typst-packages` is discarded on the next run.
Edit in this repository; the checkout only ever mirrors it.

**The script copies a whitelist, not the whole folder.** `PKG_FILES` lists what
belongs to the package; `publish.sh` and `PUBLISHING.md` are project tooling and
never reach the submission.

**The example PDFs and preview images are committed but excluded.** `exclude`
in `typst.toml` keeps them out of the bundle the compiler downloads — that is
60 KiB rather than 1 MiB — while Universe still serves them so the README can
link to and display them. Deleting them from the repository would leave dead
links on the package page, which is why step 3 resolves every link.

---

## When a check fails

| Message | Fix |
| --- | --- |
| `typst X is older than the declared compiler` | `sudo pacman -Syu typst`, or lower `compiler` in `typst.toml` if you have tested an older version |
| `gh is not authenticated` | `gh auth login` |
| `none of Arial/Helvetica/Liberation Sans is installed` | `sudo pacman -S ttf-liberation`. A warning in `--check`/`--dry-run`, a hard stop in full mode: the thumbnail is rendered here and published as-is, and a serif one can only be corrected by releasing a new version. `SKIP_FONT_CHECK=1` overrides. |
| `no typst.toml found near …` | run it from the package directory or the archive root, or set `SRC_DIR=/path/to/the/package` |
| `template/main.typ uses a relative import` | it must import `@preview/azubinachweis:0.1.0`; a relative import is an automatic rejection |
| `examples/x.typ → 2 pages` | tighten it in this order: `kopfspalten: 2`, then `luft: 0.6cm`, then smaller `mindesthoehe`/`unterschrifthoehe`, then `schriftgroesse: 9.5pt`. `weekly-report.typ` is the dense one and already spends the first four; a new compiler release changing line metrics is the usual reason this appears |
| `README.md has dead links` | a linked file was renamed or deleted; fix the path or restore the file. A link to a *directory* counts as dead too: it works locally and on GitHub but not on Universe, which serves files only — link the repository by URL instead |
| `thumbnail long edge is N` | raise `--ppi`; 250 gives roughly 2000×2900 for A4 |
| `other files changed` | the checkout has edits outside `packages/preview/azubinachweis`; a PR must contain only your package. `rm -rf ~/projects/typst-packages` and let the script clone it again |
| `already committed upstream` | the version exists on `upstream/main`. Published versions are never changed — bump to a new one, see below |
| `could not clone your fork` | `gh` is active as a different account than `GH_USER`, so `gh repo fork` forked to that account while the clone looked under `GH_USER`. Check with `gh api user --jq .login`, fix with `gh auth switch --user AyloRyd` |
| `push failed` | the branch exists on your fork from an earlier submission and has diverged. Compare, then `git -C ~/projects/typst-packages push --force-with-lease origin azubinachweis-0.1.0` — an open pull request picks the force-push up by itself |

---

## Name check before you submit

Package names may **not** be the obvious or canonical name for their function.
The guidelines' own example: `slides` is forbidden, `sliding` or `slitastic`
are fine. Names may not contain "typst", and multi-word names use `kebab-case`.

| Name | Verdict |
| --- | --- |
| `ausbildungsnachweis` | **Very likely rejected.** It is exactly the canonical term for this document — the German equivalent of `slides`. |
| `berichtsheft` | The same: the other common name for it. |
| `azubinachweis` | Chosen. Not an established term but a coined compound of "Azubi" and "Nachweis" — recognisable without claiming the search term. |

If a reviewer still finds it too close, `wochenwerk`, `lehrjahr` or `heftfuchs`
clear the rule. Renaming before the merge is cheap, but it happens here, not in
the checkout — renaming there would be undone by the next run:

```sh
sed -i 's/azubinachweis/NEW_NAME/g' typst.toml lib.typ README.md README.de.md \
  template/main.typ examples/*.typ
sed -i 's/^PKG_NAME="azubinachweis"/PKG_NAME="NEW_NAME"/' publish.sh
grep -rn azubinachweis typst.toml lib.typ README.md README.de.md template examples
```

That last grep must come back empty. Then drop the checkout — its sparse
pattern is pinned to the old name and would leave the new directory untracked:

```sh
rm -rf ~/projects/typst-packages
```

Work up through the three modes again. The new name gets its own branch and its
own pull request, so close the old one; the branch name follows `PKG_NAME`, and
a pull request cannot be moved to a different branch.

After the merge it is not possible — published packages are never renamed or
removed.

---

## After the merge

The package works as soon as the PR is merged and CI has run. It can take up to
30 minutes to appear on [Typst Universe](https://typst.app/universe/).

```sh
gh pr checks 5919 --repo typst/packages --watch    # 0.1.0; use your own number
rm -rf ~/.local/share/typst/packages/preview/azubinachweis/0.1.0
```

The number is required from outside the checkout, and the script prints the
pull request URL when it finishes. Removing the local copy makes sure you are
using the published package and not your working tree — step 4 installs it
there on every run, so it will be back after the next one.

## Releasing a new version

Submitted packages are never changed or removed; corrections ship as a new
version beside the old one — even a typo in the README costs a version bump.

Everything happens in **this** repository. The checkout under
`~/projects/typst-packages` is scratch space: step 8 recreates the branch from
`upstream/main` and copies the package in from here on every run, so anything
edited there by hand is discarded before the commit.

Fix whatever needs fixing, then bump the version. It appears in ten places:
`version` in `typst.toml`, plus nine `@preview/azubinachweis:<version>` imports
across `template/main.typ`, the six examples and both READMEs.

```sh
sed -i 's/^version = "0.1.0"/version = "0.1.1"/' typst.toml
grep -rl 'azubinachweis:0\.1\.0' . --include='*.typ' --include='*.md' \
  | xargs sed -i 's/azubinachweis:0\.1\.0/azubinachweis:0.1.1/g'
grep -rn 'azubinachweis:0\.1\.' . --include='*.typ' --include='*.md'   # check
```

Then set `PKG_VERSION="0.1.1"` at the top of `publish.sh` and work up through
the three modes again. Step 3 compares every version reference against
`PKG_VERSION` and fails if one was missed, so a half-done bump cannot reach a
pull request. The submission lands on its own branch `azubinachweis-0.1.1` as a
separate pull request; 0.1.0 stays where it is.

Two things that do not change. `@preview/azubinachweis:0.1.0` keeps working for
everyone who already imports it — users move to a new version by editing that
line themselves, nothing is pushed on them. And updates must be submitted by
whoever submitted the previous version.

---

## Sources

- [Package submission guidelines](https://github.com/typst/packages/blob/main/docs/README.md)
- [Writing a package manifest](https://github.com/typst/packages/blob/main/docs/manifest.md)
- [Tips for package authors](https://github.com/typst/packages/blob/main/docs/tips.md)
- [Local packages](https://github.com/typst/packages#local-packages)

---

## The script

It lives at the root of this repository, next to `typst.toml`. It also works
one level above a `<name>/<version>/` folder, the way a distribution archive is
laid out — it finds the package either way. If you copy it somewhere new,
`chmod +x publish.sh`.

```bash
#!/usr/bin/env bash
# =============================================================================
# publish.sh — submit azubinachweis to Typst Universe (Arch Linux)
#
#   ./publish.sh --check     validate only, touch nothing remote   (start here)
#   ./publish.sh --dry-run   do everything locally, no push, no PR
#   ./publish.sh             the whole way, asks once before pushing
#
# Every step prints PASS / FAIL. The script stops at the first failure and
# tells you what to fix.
# =============================================================================

set -Eeuo pipefail

# ---------------------------------------------------------------- settings --
GH_USER="AyloRyd"
PKG_NAME="azubinachweis"
PKG_VERSION="0.1.0"
WORK_DIR="${WORK_DIR:-$HOME/projects/typst-packages}"
DATA_HOME="${XDG_DATA_HOME:-$HOME/.local/share}"
SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"

# Files that belong to the published package. publish.sh and PUBLISHING.md are
# project tooling and deliberately stay out of it.
PKG_FILES=(typst.toml lib.typ LICENSE README.md README.de.md template examples images thumbnail.png)

# ------------------------------------------------------------------ output --
if [[ -t 1 ]]; then
  R=$'\e[31m'; G=$'\e[32m'; Y=$'\e[33m'; B=$'\e[1m'; N=$'\e[0m'
else
  R=''; G=''; Y=''; B=''; N=''
fi
STEP=0
FAILED=0
step()  { STEP=$((STEP + 1)); printf '\n%s[%d] %s%s\n' "$B" "$STEP" "$1" "$N"; }
pass()  { printf '  %sPASS%s %s\n' "$G" "$N" "$1"; }
fail()  { printf '  %sFAIL%s %s\n' "$R" "$N" "$1"; FAILED=1; }
warn()  { printf '  %sWARN%s %s\n' "$Y" "$N" "$1"; }
info()  { printf '       %s\n' "$1"; }
die()   { printf '\n%sAborted:%s %s\n' "$R" "$N" "$1" >&2; exit 1; }

trap 'printf "\n%sScript failed on line %s.%s Nothing was pushed.\n" "$R" "$LINENO" "$N"' ERR

MODE="full"
case "${1:-}" in
  --check)   MODE="check" ;;
  --dry-run) MODE="dry" ;;
  "")        MODE="full" ;;
  *)         die "unknown option '$1' — use --check, --dry-run, or no argument" ;;
esac

printf '%s=== %s %s → Typst Universe (mode: %s) ===%s\n' "$B" "$PKG_NAME" "$PKG_VERSION" "$MODE" "$N"

# =============================================================================
step "Locate the package"
# =============================================================================
# The package is the directory holding typst.toml. The script may sit next to
# it or a couple of levels above (a distribution archive keeps the tooling at
# the top and the package under <name>/<version>/), so look in both places
# rather than assuming. Everything below depends on this, so resolve it first.
if [[ -n "${SRC_DIR:-}" ]]; then
  [[ -f "$SRC_DIR/typst.toml" ]] \
    || die "SRC_DIR is set to '$SRC_DIR' but there is no typst.toml there"
else
  for candidate in \
    "$SCRIPT_DIR" \
    "$SCRIPT_DIR/$PKG_NAME/$PKG_VERSION" \
    "$SCRIPT_DIR/../$PKG_NAME/$PKG_VERSION" \
    "$SCRIPT_DIR/.."
  do
    if [[ -f "$candidate/typst.toml" ]]; then
      SRC_DIR="$(cd "$candidate" && pwd)"
      break
    fi
  done
fi
if [[ -z "${SRC_DIR:-}" ]]; then
  fail "no typst.toml found near $SCRIPT_DIR"
  info "looked in:  ."
  info "            $PKG_NAME/$PKG_VERSION"
  info "            ../$PKG_NAME/$PKG_VERSION"
  info "            .."
  die "run the script from the package directory, or set SRC_DIR=/path/to/$PKG_VERSION"
fi
pass "package found at $SRC_DIR"

# Fail early and clearly if the manifest is not the one this script targets
FOUND_NAME="$(grep -oP '^name\s*=\s*"\K[^"]+' "$SRC_DIR/typst.toml" 2>/dev/null || true)"
FOUND_VER="$(grep -oP '^version\s*=\s*"\K[^"]+' "$SRC_DIR/typst.toml" 2>/dev/null || true)"
if [[ "$FOUND_NAME" != "$PKG_NAME" || "$FOUND_VER" != "$PKG_VERSION" ]]; then
  die "found $FOUND_NAME:$FOUND_VER but this script is set up for $PKG_NAME:$PKG_VERSION.
  Edit PKG_NAME/PKG_VERSION at the top of the script, or point SRC_DIR elsewhere."
fi
pass "manifest is $FOUND_NAME:$FOUND_VER"
info "workdir:  $WORK_DIR"

# =============================================================================
step "Tools"
# =============================================================================
MISSING=()
for tool in typst git gh oxipng python3; do
  command -v "$tool" >/dev/null 2>&1 || MISSING+=("$tool")
done
if ((${#MISSING[@]})); then
  fail "missing: ${MISSING[*]}"
  info "sudo pacman -S --needed typst git github-cli oxipng ttf-liberation"
  die "install the tools above, then run again"
fi
pass "typst, git, gh, oxipng, python3 present"

TYPST_VER="$(typst --version | grep -oE '[0-9]+\.[0-9]+\.[0-9]+' | head -1)"
NEED_VER="$(grep -oP '^compiler\s*=\s*"\K[^"]+' "$SRC_DIR/typst.toml" 2>/dev/null || echo "0.12.0")"
if [[ "$(printf '%s\n%s\n' "$NEED_VER" "$TYPST_VER" | sort -V | head -1)" == "$NEED_VER" ]]; then
  pass "typst $TYPST_VER satisfies compiler = $NEED_VER"
else
  fail "typst $TYPST_VER is older than the declared compiler = $NEED_VER"
fi

# fc-list is captured first: piping it straight into `grep -q` makes grep exit
# on the first match, fc-list dies of SIGPIPE, and `pipefail` turns the whole
# pipeline into a failure — the check would report every font as missing.
FONT_LIST="$(fc-list 2>/dev/null || true)"
if grep -qiE 'Arial|Liberation Sans|Helvetica' <<<"$FONT_LIST"; then
  pass "a sans-serif from the font stack is installed"
elif [[ "$MODE" == "full" && -z "${SKIP_FONT_CHECK:-}" ]]; then
  # The thumbnail is rendered here and published as-is. Without one of these
  # fonts Typst falls back to Libertinus Serif, and the package page would show
  # a serif thumbnail that no later edit can fix — only a new version can.
  fail "none of Arial/Helvetica/Liberation Sans is installed"
  info "The thumbnail would be rendered in Libertinus Serif and published that way."
  info "  sudo pacman -S ttf-liberation"
  info "Set SKIP_FONT_CHECK=1 to publish anyway."
  die "install a matching font, then run again"
else
  warn "none of Arial/Helvetica/Liberation Sans found — output falls back to"
  info "Libertinus Serif, so the thumbnail will not look as intended."
  info "  sudo pacman -S ttf-liberation"
fi

if gh auth status >/dev/null 2>&1; then
  pass "gh is authenticated"
else
  fail "gh is not authenticated"
  info "run: gh auth login"
  die "authenticate with GitHub, then run again"
fi

# =============================================================================
step "Package contents"
# =============================================================================
for f in "${PKG_FILES[@]}"; do
  if [[ -e "$SRC_DIR/$f" ]]; then pass "$f"; else fail "$f is missing"; fi
done

MANIFEST_NAME="$(grep -oP '^name\s*=\s*"\K[^"]+' "$SRC_DIR/typst.toml")"
MANIFEST_VER="$(grep -oP '^version\s*=\s*"\K[^"]+' "$SRC_DIR/typst.toml")"
[[ "$MANIFEST_NAME" == "$PKG_NAME" ]] \
  && pass "manifest name = $MANIFEST_NAME" \
  || fail "manifest name is '$MANIFEST_NAME', expected '$PKG_NAME'"
[[ "$MANIFEST_VER" == "$PKG_VERSION" ]] \
  && pass "manifest version = $MANIFEST_VER" \
  || fail "manifest version is '$MANIFEST_VER', expected '$PKG_VERSION'"

for field in entrypoint authors license description categories; do
  grep -qE "^$field\s*=" "$SRC_DIR/typst.toml" \
    && pass "manifest has $field" \
    || fail "manifest is missing $field"
done
grep -q '^\[template\]' "$SRC_DIR/typst.toml" \
  && pass "manifest has [template]" \
  || fail "manifest is missing the [template] section"

# The template must import the package by name, never by relative path
if grep -q "@preview/$PKG_NAME:$PKG_VERSION" "$SRC_DIR/template/main.typ"; then
  pass "template/main.typ imports @preview/$PKG_NAME:$PKG_VERSION"
else
  fail "template/main.typ does not import @preview/$PKG_NAME:$PKG_VERSION"
fi
if grep -qE '\.\./(lib|.*\.typ)' "$SRC_DIR/template/main.typ"; then
  fail "template/main.typ uses a relative import — submissions are rejected for this"
else
  pass "template/main.typ has no relative import"
fi

# Every version reference must point at this version
STALE="$(grep -rl "$PKG_NAME:[0-9]" "$SRC_DIR" --include='*.typ' --include='*.md' 2>/dev/null \
         | xargs -r grep -l "$PKG_NAME:" 2>/dev/null \
         | xargs -r grep -L "$PKG_NAME:$PKG_VERSION" 2>/dev/null || true)"
if [[ -z "$STALE" ]]; then
  pass "all version references are $PKG_VERSION"
else
  fail "these files reference another version:"
  printf '       %s\n' $STALE
fi

# Both READMEs are rendered with relative links; a dead one is visible on
# Universe, so resolve every link and image target against the package. A
# directory target counts as dead: it resolves locally and on GitHub but not on
# Universe, which serves files only — link the repository by URL instead.
for rd in README.md README.de.md; do
  BROKEN="$(python3 - "$SRC_DIR" "$rd" <<'PY'
import os, re, sys
root, name = sys.argv[1], sys.argv[2]
text = open(os.path.join(root, name), encoding='utf8').read()
targets = re.findall(r'!?\[[^\]]*\]\((?!https?:)([^)#]+)\)', text)
missing = sorted({t for t in targets if not os.path.isfile(os.path.join(root, t))})
print('\n'.join(missing))
PY
)"
  if [[ -z "$BROKEN" ]]; then
    pass "$rd — every link and image resolves"
  else
    fail "$rd has dead links:"
    printf '%s\n' "$BROKEN" | sed 's/^/       /'
  fi
done

# License file must match the manifest
LIC="$(grep -oP '^license\s*=\s*"\K[^"]+' "$SRC_DIR/typst.toml")"
if grep -qi "${LIC%% *}" "$SRC_DIR/LICENSE"; then
  pass "LICENSE matches license = $LIC"
else
  fail "LICENSE does not mention '$LIC'"
fi

if (( FAILED )); then die "fix the failures above before continuing"; fi

# Copy the package into the local preview tree, so that `@preview/<pkg>:<ver>`
# resolves to this working copy. Needed twice: the examples import the package
# by that name and cannot compile without it, and step 5 reinstalls afterwards
# so the version under test also carries the freshly built examples/ and
# images/.
install_local() {
  LOCAL_PKG="$DATA_HOME/typst/packages/preview/$PKG_NAME/$PKG_VERSION"
  mkdir -p "$DATA_HOME/typst/packages/preview"
  rm -rf "$LOCAL_PKG"
  mkdir -p "$LOCAL_PKG"
  for f in "${PKG_FILES[@]}"; do cp -r "$SRC_DIR/$f" "$LOCAL_PKG/"; done
}

# =============================================================================
step "Build examples and preview images"
# =============================================================================
cd "$SRC_DIR"
mkdir -p images
install_local
pass "package staged at $LOCAL_PKG so the examples can import @preview/$PKG_NAME:$PKG_VERSION"
for f in examples/*.typ; do
  name="$(basename "${f%.typ}")"
  typst compile --root . "$f" "examples/$name.pdf" \
    || die "examples/$name.typ does not compile"
  pages="$(python3 -c "
import sys,re
d=open(sys.argv[1],'rb').read()
print(max(len(re.findall(rb'/Type\s*/Page[^s]', d)), 1))
" "examples/$name.pdf")"
  if [[ "$pages" == "1" ]]; then
    pass "examples/$name.typ → 1 page"
  else
    fail "examples/$name.typ → $pages pages (must fit on one)"
  fi
  typst compile -f png --pages 1 --ppi 96 --root . "$f" "images/$name.png" \
    || die "could not render images/$name.png"
done
oxipng -q -o 6 --strip safe images/*.png >/dev/null 2>&1 || true
pass "preview images written to images/ and optimised"

if (( FAILED )); then die "fix the failures above before continuing"; fi

# =============================================================================
step "Install the package locally and test it as a user would"
# =============================================================================
install_local
pass "installed to $LOCAL_PKG"

TESTDIR="$(mktemp -d)"
trap 'rm -rf "$TESTDIR"' EXIT
( cd "$TESTDIR" && typst init "@preview/$PKG_NAME:$PKG_VERSION" probe >/dev/null ) \
  || die "typst init failed — the [template] section or template/ folder is wrong"
pass "typst init @preview/$PKG_NAME:$PKG_VERSION works"

( cd "$TESTDIR/probe" && typst compile main.typ ) \
  || die "the freshly initialised template does not compile"
TPAGES="$(python3 -c "
import sys,re
d=open(sys.argv[1],'rb').read()
print(max(len(re.findall(rb'/Type\s*/Page[^s]', d)), 1))
" "$TESTDIR/probe/main.pdf")"
[[ "$TPAGES" == "1" ]] \
  && pass "initialised template compiles to 1 page" \
  || fail "initialised template compiles to $TPAGES pages"

# =============================================================================
step "Thumbnail"
# =============================================================================
( cd "$TESTDIR/probe" && typst compile -f png --pages 1 --ppi 250 main.typ thumbnail.png )
oxipng -q -o 4 --strip safe "$TESTDIR/probe/thumbnail.png" >/dev/null 2>&1 || true
cp "$TESTDIR/probe/thumbnail.png" "$SRC_DIR/thumbnail.png"

read -r TW TH < <(python3 -c "
import struct,sys
d=open(sys.argv[1],'rb').read(33)
w,h=struct.unpack('>II', d[16:24]); print(w,h)
" "$SRC_DIR/thumbnail.png")
TBYTES="$(stat -c%s "$SRC_DIR/thumbnail.png")"
LONG=$(( TW > TH ? TW : TH ))

(( LONG >= 1080 )) \
  && pass "thumbnail ${TW}x${TH} — long edge $LONG ≥ 1080" \
  || fail "thumbnail long edge is $LONG, must be ≥ 1080"
(( TBYTES <= 3145728 )) \
  && pass "thumbnail $((TBYTES / 1024)) KiB ≤ 3 MiB" \
  || fail "thumbnail is $((TBYTES / 1024)) KiB, must be ≤ 3 MiB"

# Only the shipped docs matter here; PUBLISHING.md is tooling and stays out of
# the package, so it is free to talk about the thumbnail.
if grep -q 'thumbnail\.png' "$SRC_DIR/README.md" "$SRC_DIR/README.de.md" 2>/dev/null; then
  fail "thumbnail.png is referenced in a README — it must not be"
else
  pass "thumbnail.png is not referenced inside the package"
fi

if (( FAILED )); then die "fix the failures above before continuing"; fi

if [[ "$MODE" == "check" ]]; then
  printf '\n%sAll checks passed.%s Nothing was changed remotely.\n' "$G" "$N"
  printf 'Run %s./publish.sh --dry-run%s next to prepare the commit without pushing.\n' "$B" "$N"
  exit 0
fi

# =============================================================================
step "Fork and clone typst/packages"
# =============================================================================
if [[ -d "$WORK_DIR/.git" ]]; then
  pass "using existing checkout at $WORK_DIR"
  git -C "$WORK_DIR" fetch --quiet upstream 2>/dev/null \
    || git -C "$WORK_DIR" remote add upstream https://github.com/typst/packages.git
else
  gh repo fork typst/packages --clone=false >/dev/null 2>&1 \
    || info "fork already exists, continuing"
  mkdir -p "$(dirname "$WORK_DIR")"
  git clone --filter=blob:none --sparse \
    "https://github.com/$GH_USER/packages.git" "$WORK_DIR" --quiet \
    || die "could not clone your fork — is github.com/$GH_USER/packages there?"
  git -C "$WORK_DIR" sparse-checkout set --no-cone \
    '/*' '!/packages' "/packages/preview/$PKG_NAME"
  git -C "$WORK_DIR" remote add upstream https://github.com/typst/packages.git
  pass "cloned your fork to $WORK_DIR (sparse)"
fi

# =============================================================================
step "Copy the package into the repository"
# =============================================================================
DEST="$WORK_DIR/packages/preview/$PKG_NAME/$PKG_VERSION"
# Ask upstream, not HEAD: after a --dry-run the local branch already carries this
# very commit, and checking HEAD would report the version as published and abort
# every run that follows a dry run.
git -C "$WORK_DIR" fetch --quiet upstream 2>/dev/null || true
if [[ -n "$(git -C "$WORK_DIR" log --oneline -1 upstream/main -- "$DEST" 2>/dev/null)" ]]; then
  die "$PKG_NAME:$PKG_VERSION is already committed upstream. Published versions are
  never changed — bump to a new version instead (see PUBLISHING.md)."
fi

# Start every run from a clean upstream/main. Without this a second run stacks
# another commit that only churns the regenerated PDFs — Typst writes a fresh
# document id each time — and the pull request would carry both. The branch is
# recreated before the package is copied in, so the checkout cannot delete the
# fresh copy.
BRANCH="$PKG_NAME-$PKG_VERSION"
git -C "$WORK_DIR" checkout -q -B "$BRANCH" upstream/main

rm -rf "$DEST"
mkdir -p "$DEST"
for f in "${PKG_FILES[@]}"; do cp -r "$SRC_DIR/$f" "$DEST/"; done
# The example PDFs and preview images stay in the repository even though
# `exclude` keeps them out of the downloaded bundle: Universe serves them so
# the README can link to them. Removing them here would leave dead links.
pass "copied $(printf '%s ' "${PKG_FILES[@]}")to $DEST"
info "$(find "$DEST" -type f | wc -l) files, $(du -sh "$DEST" | cut -f1) total"

# -uall lists untracked files individually; without it git collapses a new
# directory to a single "?? packages/" entry and the filter below misses it.
OTHER="$(git -C "$WORK_DIR" status --porcelain -uall \
         | grep -v "packages/preview/$PKG_NAME/" || true)"
if [[ -z "$OTHER" ]]; then
  pass "working tree touches only packages/preview/$PKG_NAME"
else
  fail "other files changed — a PR must contain only your package:"
  printf '%s\n' "$OTHER" | sed 's/^/       /'
  die "clean the working tree and run again"
fi

# =============================================================================
step "Commit"
# =============================================================================
cd "$WORK_DIR"
git add "packages/preview/$PKG_NAME"
if git diff --cached --quiet; then
  warn "nothing to commit — the files are already staged or identical"
else
  git commit -q -m "Add $PKG_NAME:$PKG_VERSION"
  pass "committed on branch $BRANCH"
fi
info "$(git show --stat --oneline HEAD | head -3)"

if [[ "$MODE" == "dry" ]]; then
  printf '\n%sDry run complete.%s Everything is ready in %s\n' "$G" "$N" "$WORK_DIR"
  printf 'Nothing was pushed. Review with %sgit -C %s show%s, then run %s./publish.sh%s.\n' \
    "$B" "$WORK_DIR" "$N" "$B" "$N"
  exit 0
fi

# =============================================================================
step "Push and open the pull request"
# =============================================================================
printf '\n  This pushes to github.com/%s/packages and opens a PR against\n' "$GH_USER"
printf '  typst/packages. Submitted packages are never removed or renamed.\n'
read -r -p "  Continue? [y/N] " REPLY
[[ "$REPLY" =~ ^[Yy]$ ]] || die "stopped at your request — nothing was pushed"

git push -u origin "$BRANCH" --quiet || die "push failed"
pass "pushed $BRANCH"

PR_URL="$(gh pr create --repo typst/packages \
  --title "Add $PKG_NAME:$PKG_VERSION" \
  --body "Ausbildungsnachweis (German apprenticeship training record) as a weekly report, daily report or cover sheet, each on a single page.

Tested locally with \`typst init @preview/$PKG_NAME:$PKG_VERSION\`; the initialised template and all examples compile to one page each." \
  2>&1 | tail -1)"
pass "pull request opened"
info "$PR_URL"

# =============================================================================
printf '\n%s=== Done ===%s\n' "$B" "$N"
printf 'Watch CI:        gh pr checks --repo typst/packages\n'
printf 'After the merge it can take up to 30 minutes to appear on Typst Universe.\n'
printf 'Then drop the local install so you use the published copy:\n'
printf '  rm -rf %s\n' "$LOCAL_PKG"
```
