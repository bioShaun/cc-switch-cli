# cc-switch: collapse `add` + `use` into one step

## Context

User finds `cc-switch add <name> <file> && cc-switch use <name>` cumbersome. Goal: extend `use` so it accepts the same argument shapes as `add` and auto-imports the profile when needed, so a single `cc-switch use …` covers the common case. `add` and `use` keep their current behavior for back-compat; the simplification is purely additive.

End state: `cc-switch use foo.json`, `cc-switch use work ~/.claude/settings.work.json`, and `cc-switch use gpt` (when `gpt.json` lives in a known scan location but not yet in `profiles/`) all work as one command. Existing `cc-switch use <existing-profile>` continues to behave identically.

## Approach

Single-file change in `cc-switch` plus a published copy of this plan under `docs/`. Steps are sequential within the `use` case body; step 5 (usage/help), step 6 (README), and step 7 (docs publish) are independent of 1–4 and of each other.

### 1. Loosen `use` arg count

In the `use)` case (currently line 195), replace the `[[ $# -eq 1 ]]` guard with `[[ $# -eq 1 || $# -eq 2 ]]`, and update the inline usage string in the failure branch to:

```
Usage: cc-switch use <profile> | cc-switch use <profile> <settings.json|profile-name> | cc-switch use <settings.json>
```

### 2. Resolve profile name + source the same way `add` does

Replace the current `profile=$1; src=${profiles}/${profile}.json; [[ -f "$src" ]] || …` block (lines 197–199) with the same resolution `add` performs (lines 224–230), but tolerate the "already a registered profile" path without reporting it as an import:

```bash
imported=0
if [[ $# -eq 1 ]]; then
  arg=$1
  # Fast path: bare name that already exists as a profile — no import, no scan.
  if [[ "$arg" != */* && "$arg" != *.json && -f "${profiles}/${arg}.json" ]]; then
    profile=$arg
    src=${profiles}/${profile}.json
  else
    src=$(source_from_arg "$arg") || { echo "Settings file or profile not found: $arg" >&2; exit 1; }
    profile=$(profile_name_from_file "$src")
  fi
else
  profile=$1
  src=$(source_from_arg "$2") || { echo "Settings file or profile not found: $2" >&2; exit 1; }
fi

[[ "$profile" != *'/'* && "$profile" != *'.'* ]] || { echo 'Profile name must not contain / or .' >&2; exit 2; }
json_check "$src" || { echo "Invalid JSON: $src" >&2; exit 1; }
```

The fast-path branch matters: without it, `cc-switch use work` (registered profile) would route through `source_from_arg` → `resolve_named_source`, which returns `${profiles}/${profile}.json` and works, but `profile_name_from_file` strips a `settings.` prefix that the saved profile name will not have, so the round-trip is a no-op for normal names. Still, the explicit fast path keeps `use <existing>` byte-for-byte identical to today's behavior and avoids surprising rename if a user ever named a profile `settings.foo`.

### 3. Auto-import into `profiles/` when needed

Immediately after the resolution block in step 2, before the regular-file refusal check, add:

```bash
mkdir -p "$profiles"
dest=${profiles}/${profile}.json
if [[ "$(abs_path "$src")" != "$(abs_path "$dest")" ]]; then
  cp "$src" "$dest"
  imported=1
fi
src=$dest
```

This mirrors `add` (lines 234–237). After it runs, `src` always points inside `$profiles`, so the rest of the existing `use` body (regular-file guard at 202–207, copy/symlink at 209–217) operates unchanged.

### 4. Adjust the success message

Replace the two `echo "Switched …"` / `echo "Copied …"` lines (213, 216) so they prepend a one-line import notice when `imported=1`:

```bash
[[ $imported -eq 1 ]] && echo "Imported profile: $dest"
```

placed once just before the `if [[ "$mode" == copy ]]; then` block. Keep the existing post-switch echoes verbatim. Order: import line first, then switch line — so output reads like two distinct facts when both happen.

### 5. Update `usage()`

In the heredoc (lines 5–37):

- Replace the `cc-switch [options] use <profile>` line with three lines mirroring `add`:
  ```
  cc-switch [options] use <profile>
  cc-switch [options] use <profile> [settings.json|profile-name]
  cc-switch [options] use <settings.json>
  ```
- Append two examples to the global Examples block (after the existing `cc-switch current` line):
  ```
  cc-switch use ~/.claude/settings.work.json   # add + activate in one step
  cc-switch use gpt ~/.claude/settings.gpt.json
  ```
- Append one example to the project examples block (after `cc-switch --project use dev`):
  ```
  cc-switch --project use dev .claude/settings.dev.json
  ```

### 6. Update README

In `README.md`, after the existing `cc-switch use default` / `cc-switch use work` block (around line 44), insert a short Chinese section titled `## 一步切换`:

```
也可以一步完成 add + use：

\`\`\`bash
# 直接传 JSON 文件，profile 名自动从文件名推导
cc-switch use ~/.claude/settings.work.json

# 显式指定 profile 名 + 源文件
cc-switch use gpt ~/.claude/settings.gpt.json

# --project 同样支持
cc-switch --project use dev .claude/settings.dev.json
\`\`\`

如果对应的 profile 已存在，行为与 \`cc-switch use <name>\` 完全一致；否则会先导入到 profiles 目录再切换。
```

(Use real backticks in the file; escaped here only for plan readability.)

### 7. Publish this plan under `docs/`

The user requested the plan live in `docs/`, not only in the ephemeral session location. After implementing steps 1–6, copy this plan file verbatim to the repo:

```bash
mkdir -p docs
cp "$(pwd)/local/cc-switch-use-auto-add-plan.md" docs/cc-switch-use-auto-add-plan.md  # if running from session root
```

The implementer's actual source path for this plan file is whichever `local://cc-switch-use-auto-add-plan.md` resolves to in their session — most reliable command:

```bash
mkdir -p docs
# Replace <SESSION_PLAN_PATH> with the resolved filesystem path of local://cc-switch-use-auto-add-plan.md
cp <SESSION_PLAN_PATH> docs/cc-switch-use-auto-add-plan.md
```

If the running agent has no session-relative copy available (e.g. fresh-context execution mode wiped it), recreate `docs/cc-switch-use-auto-add-plan.md` by writing the full content of the plan the agent is currently executing — the file in `docs/` MUST byte-match the plan being executed, including this step 7.

No other files in `docs/` are touched. Do not add a `docs/README.md` index or any other content.

## Critical files & anchors

- `cc-switch` lines 194–218 — the `use)` case; all behavior changes live here.
- `cc-switch` lines 220–239 — the `add)` case; reference for the resolution + copy pattern being mirrored.
- `cc-switch` lines 67–105 — `profile_name_from_file`, `resolve_named_source`, `source_from_arg`; reused as-is, no changes.
- `cc-switch` lines 4–38 — `usage()` heredoc updated in step 5.
- `README.md` lines 41–46 — insertion point for the new "一步切换" section in step 6.

## Verification

Run from the repo root. Use a throwaway HOME so global state is not touched:

```bash
export TMPHOME=$(mktemp -d)
mkdir -p "$TMPHOME/.claude"
printf '{"a":1}\n' > "$TMPHOME/.claude/settings.work.json"
printf '{"a":2}\n' > "$TMPHOME/.claude/settings.gpt.json"
HOME=$TMPHOME ./cc-switch use "$TMPHOME/.claude/settings.work.json"
HOME=$TMPHOME ./cc-switch list           # expect: * work
HOME=$TMPHOME ./cc-switch current        # expect: work
HOME=$TMPHOME ./cc-switch use gpt "$TMPHOME/.claude/settings.gpt.json"
HOME=$TMPHOME ./cc-switch list           # expect: gpt with *, work without
HOME=$TMPHOME ./cc-switch use work       # back-compat: existing profile name
HOME=$TMPHOME ./cc-switch current        # expect: work
HOME=$TMPHOME ./cc-switch use nope 2>&1  # expect non-zero, "Settings file or profile not found: nope"
```

Project-mode smoke test:

```bash
WORK=$(mktemp -d); cd "$WORK"
mkdir -p .claude
printf '{"x":1}\n' > .claude/settings.dev.json
HOME=$TMPHOME /home/tcuni/cc-switch-cli/cc-switch --project use dev .claude/settings.dev.json
HOME=$TMPHOME /home/tcuni/cc-switch-cli/cc-switch --project current   # expect: dev
```

Regular-file guard still fires:

```bash
rm "$TMPHOME/.claude/settings.json" 2>/dev/null || true
printf '{"y":1}\n' > "$TMPHOME/.claude/settings.json"   # plain file, not symlink
HOME=$TMPHOME ./cc-switch use "$TMPHOME/.claude/settings.work.json" 2>&1
# expect non-zero exit, "Refusing to replace regular file"
HOME=$TMPHOME ./cc-switch --copy use "$TMPHOME/.claude/settings.work.json"
# expect success, settings.json now contains {"a":1}
```

Docs publish check (covers step 7):

```bash
test -f docs/cc-switch-use-auto-add-plan.md && echo OK   # expect: OK
head -1 docs/cc-switch-use-auto-add-plan.md              # expect: # cc-switch: collapse `add` + `use` into one step
```

Cleanup: `rm -rf "$TMPHOME" "$WORK"`.

## Assumptions & contingencies

- Auto-import overwrites an existing profile of the same name without prompting (matches `add`'s current behavior — `cp` over `dest`). If reality is that the user expects a confirmation, the fix is to add an `[[ -f $dest ]] && { echo "Profile $profile already exists; refusing to overwrite. Use 'cc-switch add' explicitly." >&2; exit 1; }` guard before the `cp`; do not add this preemptively.
- `add` is kept as-is. No deprecation, no alias, no message nudging users toward `use`. The two-command flow remains valid for users who want to register profiles without activating them.
- The `imported` notice is printed to stdout (not stderr) so scripts capturing output see a stable two-line block when an import happens. If a caller depends on single-line output, they were already coupled to the exact wording and will need to update regardless.
