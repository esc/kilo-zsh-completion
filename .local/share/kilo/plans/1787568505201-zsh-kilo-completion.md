# Plan: Refinements to session completion in `_kilo`

## Context

`_kilo` (at `~/code/kilo-zsh-completion/_kilo`, v0.2 unreleased) currently
implements session completion for `-s/--session`, `session delete`,
`export`, and `cloud * --session-id` via three functions:

- `_kilo_session_pairs` (~line 215): emits `id<TAB>directory<TAB>title`
  tuples from `kilo session list --all --format json` (cached via
  `_kilo_cached`), explicitly sorted by `updated` descending (both `jq`
  and `awk`+`sort` fallback paths).
- `_kilo_session_group` (~line 254): turns ids+titles into display strings
  `"<title>  (ses_<7-chars>...)"` and calls
  `compadd -J "$tag" -V "$tag" -X "$label" -d displays -a ids`.
- `_kilo_sessions` (~line 269): collects the tuples into parallel arrays,
  then decides between two modes:
  - **Scoped**: if `$PWD` equals or is inside a session's `directory`, show
    only that directory's sessions (single group).
  - **Browse-all**: otherwise, list every session, one group per
    directory, header per group.

Menu-select is enabled by default for all kilo completions (top of `_kilo`
body, ~line 1088, `zstyle -m ... || zstyle ... menu yes select`), unless
the user has already configured a `menu` style.

## Verified facts (from `kilo session list --all --format json` on 7.5.16)

- Each session object has: `id`, `title`, `updated`, `created`,
  `projectId`, `directory`, and a nested `project` object with `id` and
  `worktree`.
- For sessions whose `projectId` is `global`, `directory` is whatever
  directory kilo happened to run in (e.g. `/home/vhaenel`,
  `/home/vhaenel/reviews`) and `project.worktree` is `/`. "global" is
  kilo's catch-all bucket for "not inside any registered project" — it is
  NOT a real project directory.
- For real projects, `directory` and `project.worktree` are identical.

## Refinement 1: Only scope when the current directory is a real project

Current bug: completing in `~` matches the `directory` value
`/home/vhaenel` on the *global* sessions, so the completion gets scoped to
just those sessions instead of showing everything grouped.

Required change in `_kilo_sessions`:

- Extend `_kilo_session_pairs` to emit `projectId` as a fourth field:
  - jq path: add `\t\(.projectId)` to the output template.
  - awk fallback: capture `projectId` like the other fields (it appears
    before `directory` in the pretty-printed JSON, so add a match rule and
    include it in the printf).
- Parse into a `pids` array alongside `ids`/`dirs`/`titles`.
- Only treat a directory as a candidate for scoping when the sessions
  under it belong to a non-`global` project. Concretely: build
  `uniq_dirs` from directories of sessions whose `pids[i] != "global"`,
  so `$PWD` matching a global session's `directory` (like `~`) does NOT
  trigger scoping — it falls through to browse-all as intended.
- Grouping in browse-all mode is unchanged (group key = `directory`; global
  sessions from different dirs just form their own groups — acceptable).

## Refinement 2: One session per line

The current `compadd -J "$tag" -V "$tag"` doesn't force a one-per-row
list layout. Add to `_kilo_session_group`'s `compadd` call:

    -l        # list matches one per line

(`compadd -l` forces the listing to show one match per line instead of
packing multiple per row.)

## Files to change

- `~/code/kilo-zsh-completion/_kilo` only:
  - `_kilo_session_pairs` — add `projectId` field to both code paths.
  - `_kilo_sessions` — parse the new field; restrict scoping candidate
    dirs to non-global projects.
  - `_kilo_session_group` — add `-l` to the `compadd` call.
  - Header docs (lines ~43-57): update the description of the scoping rule
    to say it applies only to sessions belonging to a real (non-global)
    project.
  - CHANGELOG `* v0.2 (unreleased)`: adjust the scoping bullet to mention
    the non-global restriction and the one-per-line listing.

## Validation

1. `zsh -n` syntax check.
2. Direct unit check of `_kilo_session_pairs` output (both jq and
   no-jq paths) — four tab-separated fields, recency order.
3. zpty/`_complete_debug` end-to-end traces, three cwds:
   - `cd ~/numba` → `kilo -s <TAB>` → single group, numba sessions only.
   - `cd ~` → `kilo -s <TAB>` → ALL sessions, grouped per directory
     (regression check for refinement 1).
   - `cd /tmp/kilo` (no sessions there) → browse-all, same as `~`.
   Verify `compadd` calls include `-l` and per-directory group tags.
4. Clear `~/.cache/kilo-completion/sessions` first so the new field
   actually gets fetched.

## Risks / notes

- The no-jq awk path assumes kilo's pretty-printed JSON emits
  `id`/`title`/`updated`/`projectId` before `directory` inside each object
  — verified against real 7.5.16 output; keep an eye on field order if kilo
  changes its formatter.
