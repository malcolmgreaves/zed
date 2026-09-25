# Transfer context: the "Git Commit Diff" feature

> **Do not merge this file.** It is a working handoff document for continuing
> development / repeated rebasing of the commit-diff feature. Delete before the
> PR is finalized.

---

## 0. Why this document exists

This feature lives as **a single commit on the branch `mg/git_branch_diff_any`**,
and the recurring task is to **rebase that one commit cleanly onto a fast-moving
`main`**. Between rebases, `main` changes a lot — files get renamed, the diff
subsystem gets refactored, new `DiffBase` variants appear. A purely mechanical
"resolve the conflict markers" approach breaks down, because the code the feature
originally patched keeps moving or being rewritten.

So this document explains the feature **conceptually** — what it does, and the
*intent* of every integration point — so that a future session can re-locate
where each piece belongs in whatever shape `main` has taken, and re-apply the
feature correctly rather than blindly resolving diffs.

**If you only read one thing:** the feature adds a new `DiffBase::Commit { base_ref }`
variant to Zed's diff subsystem, and wires up a UI to create/drive it. It is a
near-exact twin of the existing **branch diff** (`DiffBase::Merge`). Everywhere
the codebase handles `Merge`, commit diff needs equivalent handling for `Commit`.
**Find how `DiffBase::Merge` is implemented in the current tree, and mirror it.**

---

## 1. What the feature is (user-facing)

A **"git: commit diff"** command. It opens a diff view comparing the working tree
against an **arbitrary user-supplied git ref** — a hash, branch, tag, or revision
expression such as `HEAD~2`. It uses the **same merge-base semantics** as the
existing "branch diff" view, but the base ref is entered by the user instead of
being resolved from the default branch.

User flow:
1. User invokes the `git: commit diff` action (command palette / keybinding).
2. A **ref picker modal** (`RefPickerModal`) appears; user types a git ref. The
   modal previews the resolved commit.
3. On confirm, a diff tab opens (or the existing commit-diff tab is reused and
   retargeted) showing working-tree-vs-ref changes.
4. The tab's toolbar shows a **"Base: <ref>"** control. Clicking it re-opens the
   ref picker so the user can retarget to a different ref.

It is deliberately the commit-oriented sibling of **branch diff** (`git: branch diff`,
action struct `DeployBranchDiff`), which diffs against the default branch and whose
toolbar control is a *branch picker*.

---

## 2. Branch / workflow context

- Feature branch: `mg/git_branch_diff_any`, tracking remote `mg/mg/git_branch_diff_any`.
- The feature is **one commit** titled `git diff commit view` (SHA churns every
  rebase; was `e18902f224` → `5d18f5dcf0` → `e2c96ccab4` → `ce08b72095` → `bc96646f16` …).
- The recurring request: "main updated, rebase again cleanly, don't lose the
  feature and don't clobber main's changes."
- **After rebasing, the branch diverges from its upstream** (expected). Do **not**
  push unless asked; the user handles `git push --force-with-lease`.
- A stray `RESUME` file (containing `claude --resume <session-id>`) was once
  accidentally committed. It is **not** part of the feature — keep it out of the
  commit.
- Safety backups are created before each rebase, named
  `backup-mg-git_branch_diff_any-<shortsha>[-<epoch>]`. Make one before you start.

---

## 3. Conceptual architecture of Zed's diff subsystem

This is the mental model you need. Names/paths are current as of this writing but
**will drift** — the concepts are what matter.

### 3.1 `DiffBase` — the core enum

Location (current): `crates/project/src/git_store/diff_buffer_list.rs`.
(Historically this file was `branch_diff.rs` in the `project` crate; `main`
renamed it. If you can't find it, `git grep "enum DiffBase"`.)

`DiffBase` names *what a diff view is comparing the working tree against*. Variants
seen over time:

- `Head` — uncommitted changes (working tree vs index/HEAD). No tree diff needed.
- `Index` — unstaged changes.
- `Staged` — staged changes.
- `Merge { base_ref }` — **branch diff**: working tree vs the merge-base of `base_ref`
  and HEAD.
- `Commit { base_ref }` — **THIS FEATURE**: working tree vs an arbitrary ref, using
  the *same* merge-base engine as `Merge`.

The enum derives `serde::Serialize/Deserialize` (used for workspace persistence),
`Clone, PartialEq, Eq, Debug`.

Two helper methods on `DiffBase` are part of the feature's design (they may or may
not exist upstream; if the feature added them, keep them):
- `requires_tree_diff() -> bool` → `true` for `Merge | Commit` (they need a committed
  tree diff against a base ref; `Head/Index/Staged` do not). *This was originally
  named `is_merge_base()` upstream and the feature renamed it to cover both variants.*
  If upstream still calls it `is_merge_base`, decide whether to rename or add
  alongside — but every caller must treat `Commit` like `Merge`.
- `base_ref() -> Option<&SharedString>` → `Some` for `Merge | Commit`, `None`
  otherwise. Handy for the toolbar and generic handling.

### 3.2 The layering (bottom → top)

```
DiffBase                     (enum: what are we diffing against)
   │
DiffBufferList               (computes the set of changed buffers + their diffs
   │                          for a given DiffBase; owns the repo + tree diff)
   │
DiffMultibuffer              (wraps a DiffBufferList into a multibuffer editor:
   │                          the actual reviewable diff surface)
   │
Item types (each wraps a DiffMultibuffer and provides the workspace tab):
   • ProjectDiff   → DiffBase::Head        (project_diff.rs)
   • UnstagedDiff  → DiffBase::Index       (unstaged_diff.rs)
   • StagedDiff    → DiffBase::Staged      (staged_diff.rs)
   • BranchDiff    → DiffBase::Merge  AND  DiffBase::Commit   (branch_diff.rs)   ← feature lives here
```

**Critical architectural fact:** as of the latest rebase, `main` extracted the
branch-diff logic into its own file `crates/git_ui/src/branch_diff.rs` with a
dedicated `BranchDiff` Item type (and `BranchDiffToolbar`). Earlier in history,
branch diff was *inside* `ProjectDiff`. **The feature has migrated with it** — the
feature now lives in `branch_diff.rs`, generalizing `BranchDiff` to also carry
`DiffBase::Commit`. If a future `main` moves branch diff again, move the feature
with it.

### 3.3 The ref picker (UI entry point)

Location (current): `crates/git_ui/src/git_ui.rs`.

- `RefPickerModal` — a modal with a text editor for a git ref + a commit preview.
- `RefPickerAction` (enum) — what the modal does on confirm:
  - `ViewCommit` → opens a read-only commit view (`CommitView::open`). *This is the
    pre-existing "view commit" behavior.*
  - `CommitDiff` → **feature**: calls `BranchDiff::deploy_commit_diff(...)`.
- `RefPickerModal::new(repo, workspace, action, window, cx)` takes the action so the
  same modal serves both. `show_ref_picker` (the "view commit" entry) passes
  `RefPickerAction::ViewCommit`.
- On failed ref resolution it shows a toast: `Couldn't resolve git ref "<ref>"`.

### 3.4 The action declaration

Location (current): `crates/git_ui/src/project_diff.rs`, inside the `actions!(git, [...])`
macro. The feature adds a `CommitDiff` action there (next to `CompareWithBranch`).
Reason it lives in `project_diff.rs`: that's where the crate declares its git diff
actions (`Diff`, `Add`, `ReviewDiff`, `CompareWithBranch`, and the `DeployBranchDiff`
struct action). `branch_diff.rs` imports `CommitDiff` from `project_diff`.

### 3.5 The toolbar

`BranchDiffToolbar` (current: in `branch_diff.rs`; installed once from
`crates/zed/src/zed.rs` via `cx.new(BranchDiffToolbar::new)`). It renders the base
selector + review/stat controls for a `BranchDiff` item. The feature makes the base
selector conditional on the variant:
- `Merge` → a `PopoverMenu` **branch picker** (`branch_picker::select_popover`).
- `Commit` → a plain `Button` labeled `Base: <ref>` that **dispatches the `CommitDiff`
  action** to re-open the ref picker.

---

## 4. The feature's full integration surface

Every place that must know about `DiffBase::Commit`. Descriptions are conceptual so
you can re-find them; current file:line is a hint, not a guarantee.

### In `crates/project/src/git_store/diff_buffer_list.rs`
1. **The `DiffBase` enum** — add the `Commit { base_ref: SharedString }` variant
   (with a doc comment noting it shares merge-base semantics with `Merge`).
2. **`requires_tree_diff()`** — `Commit` returns `true` (grouped with `Merge`).
3. **`base_ref()`** — `Commit { base_ref } => Some(base_ref)`; `Head/Index/Staged => None`.
4. **Status-merging match** (the loop over `cached_status()` that decides each file's
   status) — `Commit` behaves exactly like `Merge` (`self.merge_statuses(...)`).
5. **`load_buffer` match** (which chooses how to open each buffer's diff) — `Commit`
   joins the `Merge` arm (`open_diff_since` with the tree-diff entry).
   → Traps: adding the enum variant makes these matches **non-exhaustive**; the
   compiler (E0004) will point you at each one. There is no meaningful behavioral
   difference from `Merge` in this file.

### In `crates/git_ui/src/branch_diff.rs` (the bulk of the feature)
Mirror each `Merge` behavior for `Commit`:
1. **Imports** — bring in `RefPickerAction`, `RefPickerModal` (from crate root) and
   `CommitDiff` (from `project_diff`).
2. **`register`** — `workspace.register_action(Self::prompt_commit_diff)`.
3. **`prompt_commit_diff(workspace, &CommitDiff, window, cx)`** — opens
   `RefPickerModal::new(repo, workspace_entity, RefPickerAction::CommitDiff, …)`.
4. **`deploy_commit_diff(workspace, base_ref, window, cx)`** — `pub(crate)`. Emits a
   `telemetry::event!("Git Commit Diff Opened")`. Finds an existing `BranchDiff` item
   whose base is `DiffBase::Commit { .. }`; if found, activates it, updates its repo,
   and calls `set_commit_base(base_ref)`. Otherwise constructs a new one via
   `new_with_diff_base(DiffBase::Commit { base_ref }, …)` and adds it to the pane.
   (This mirrors `deploy_branch_diff_with_base_ref`, but is **synchronous** — no
   default-branch resolution needed since the ref is already supplied — and reuses a
   single commit-diff tab.)
5. **`new_with_diff_base(diff_base, project, workspace, repo, window, cx)`** — the
   generalized constructor. Upstream's `new_with_base_ref` hardcoded
   `DiffBase::Merge { base_ref }`; the feature refactors so `new_with_base_ref`
   delegates to `new_with_diff_base` with `Merge`, and commit diff calls it with
   `Commit`. Both variants get identical editor/addon setup (merge styling,
   `RestoreOnlyDiffHunkDelegate`, `BranchDiffAddon` for status override).
6. **`set_commit_base(base_ref, cx)`** — twin of `set_merge_base`, sets
   `DiffBase::Commit`.
7. **`tab_content_text`** — `Commit { base_ref } => "Diff vs <ref>"` (branch diff uses
   `"Changes since <ref>"`).
8. **`review_diff`** — accept `Merge | Commit` (both reviewable via
   `DiffType::MergeBase`).
9. **`clone_on_split`** — preserve the exact variant (`Merge` or `Commit`) when
   splitting; reconstruct via `new_with_diff_base` with the cloned `diff_base`.
10. **`SerializableItem` (serialize + deserialize)** — persist and restore `Commit`
    tabs too (they serialize under the same `"BranchDiff"` kind). Accept
    `Merge | Commit` in both directions.
11. **`compare_with_branch`** — its `selected_branch` match: `Commit` groups with the
    `None` arm (a commit ref is not a branch, so nothing to pre-select).
12. **`BranchDiffToolbar::render`** — build the base selector by variant (see §3.5).
    Use `diff_base.base_ref()` to guard/early-return for the non-ref bases.
13. **Test `test_commit_diff`** — mirrors `test_branch_diff` but constructs with
    `DiffBase::Commit { base_ref: "HEAD~1".into() }` and asserts the same diff output.

### In `crates/git_ui/src/git_ui.rs`
1. **`RefPickerAction`** enum (`ViewCommit`, `CommitDiff`) + `title()`.
2. **`RefPickerModal`** carries an `action` field; `new` takes it.
3. The confirm handler dispatches on `action`: `CommitDiff` →
   `crate::branch_diff::BranchDiff::deploy_commit_diff(workspace, git_ref_string.into(), …)`.
   (Fully-qualified path so it's robust to import churn — `git_ui.rs`'s imports are
   volatile.)
4. `BranchDiff::register` is already called in `git_ui::init`'s `observe_new` for
   `Workspace` (line ~97). No extra registration needed — `prompt_commit_diff` is
   picked up via `BranchDiff::register`.

### In `crates/git_ui/src/project_diff.rs`
1. Add the `CommitDiff` action to the `actions!(git, [...])` macro. That's the *only*
   change here — the rest of the feature was relocated to `branch_diff.rs`.

### Files NOT needing changes (but verify)
`staged_diff.rs`, `unstaged_diff.rs`, `diff_multibuffer.rs` reference `DiffBase` but
had no exhaustive production matches missing `Commit` (their `DiffBase` uses are
constructors or test assertions). Confirm with a compile — E0004 will flag any real
exhaustiveness break anywhere in the crate.

---

## 5. Key design decision (and its rationale)

**Commit diff reuses the `BranchDiff` Item type rather than introducing a separate
`CommitDiff` Item type.** `BranchDiff` is generalized to hold either
`DiffBase::Merge` or `DiffBase::Commit`.

Why: a commit diff is *semantically identical* to a branch diff (same merge-base
engine, same editor surface, same review flow); only the source of the base ref and
the toolbar control differ. A separate Item type would duplicate the entire `Item` +
`SerializableItem` + toolbar plumbing. This also matches the feature's *original*
design, where the single branch-diff view (then inside `ProjectDiff`) handled both.

If a reviewer prefers a dedicated `CommitDiff` type, that's a larger refactor — flag
it, don't assume it.

---

## 6. The rebase playbook (this is the recurring task)

1. **Assess:** `git fetch`, then compare branch vs `main`:
   - `git log --oneline main..HEAD` (should be the one feature commit)
   - `git log --oneline HEAD..main | wc -l` (how far behind)
   - `git merge-base HEAD main` vs `git rev-parse main`.
2. **Back up:** `git branch backup-mg-git_branch_diff_any-$(git rev-parse --short HEAD)-<epoch>`.
   (`Date.now()`-style epoch: pass a timestamp; the shell `date +%s` is fine here.)
3. **Understand how `main` changed the diff subsystem** *before* resolving. Check
   which of the feature's touch-point files main modified, and especially whether
   files were **renamed** or logic **relocated**:
   - `git log --oneline <merge-base>..main -- <each feature file>`
   - `git grep "enum DiffBase" main` / `git grep "struct BranchDiff" main` /
     `git grep "deploy_branch_diff\|DiffBase::Merge" main -- 'crates/git_ui/*.rs'`
   - Read `main`'s current `branch_diff.rs` (or wherever `Merge` now lives) — **it is
     your template**. The feature is "do what `Merge` does, for `Commit`."
4. **Start the rebase:** `git rebase main`.
5. **Resolve strategically, not mechanically.** When `main` has rewritten a file the
   feature patched (common), don't wrestle conflict markers. Instead:
   - Take `main`'s version of that file (`git checkout --ours <file>` during a rebase —
     `--ours` is `main`/HEAD, `--theirs` is the feature commit), then
   - **Re-apply the feature's intent** onto it using §4 as the checklist and `main`'s
     `Merge` implementation as the template.
   - For files with small overlaps (e.g. the enum, `git_ui.rs`), resolve markers
     directly, always **keeping both** main's and the feature's additions.
6. **Drop junk:** ensure no `RESUME` file (`git rm --cached RESUME` if present).
7. **Compile & test** (see §7) *before* `git rebase --continue`.
8. **Continue:** `GIT_EDITOR=true git rebase --continue` (keeps the commit message).
9. **Verify clean rebase:** branch is exactly 1 ahead, `HEAD..main` empty,
   `merge-base(HEAD,main) == main`, working tree clean, commit touches only the
   expected files, no conflict markers (`git grep -e '^<<<<<<< ' -e '^=======$' -e '^>>>>>>> '`).

---

## 7. Verification

- Build: `cargo check -p git_ui` (compiles `project` as a dep, so both crates are
  covered). Use `./script/clippy` for the project's lint pass if doing a fuller check.
- Tests (cargo takes ONE filter substring):
  - `cargo test -p git_ui --lib 'branch_diff::tests::'` → must include `test_commit_diff`
    passing, plus main's `test_branch_diff`, `test_branch_diff_action_matches_existing_item_by_base_ref`.
  - `cargo test -p git_ui --lib 'project_diff::tests::'` → all green.
- **Environment gotcha:** `target/` is a **symlink to an external volume**
  (`/Volumes/owc_speedy/...`). If that volume is unmounted, cargo fails with
  `Not a directory (os error 20)` / broken symlink — this is **not** a code problem.
  Either wait for the volume, or set `CARGO_TARGET_DIR=<scratch>` for a (slow) cold
  build. When the resulting files are byte-identical to a previously-green build,
  a recompile is provably redundant (identical base + identical patch ⇒ identical
  result) — but still run it if the volume is available.

---

## 8. Traps & lessons (from past rebases)

- **Adding the `Commit` variant makes many matches non-exhaustive.** Lean on the
  compiler (E0004) — it enumerates every site. But also handle the **refutable
  `let ... else` / `if let`** sites (review, split, serialize) that *don't* trigger
  E0004 but would silently make commit diffs behave wrong (§4).
- **`main` renames/relocates aggressively.** Examples that already happened:
  - `is_merge_base()` → renamed to `requires_tree_diff()` by the feature.
  - `branch_diff.rs` (project crate) → renamed to `diff_buffer_list.rs` by main.
  - Branch-diff logic → extracted from `ProjectDiff` into a new `branch_diff.rs`
    (git_ui crate) with `BranchDiff` + `BranchDiffToolbar` by main.
  - New `DiffBase::Index` / `DiffBase::Staged` variants + `StagedDiff`/`UnstagedDiff`
    item types added by main ("partially staged changes").
  Don't assume any path; `git grep` the concept.
- **`git_ui.rs` import block is volatile** (linters/users reorder it). Reference
  `BranchDiff::deploy_commit_diff` by fully-qualified `crate::branch_diff::...` path.
- **Tab title vs toolbar:** older `main` split `tab_content` (label) and a breadcrumb
  ("Diff vs X"); newer `main` folds it into `tab_content_text`. Put the commit label
  wherever `Merge`'s label currently lives.
- **Persistence:** commit-diff tabs serialize under the `"BranchDiff"` kind. If you
  skip the serialize/deserialize arms, commit tabs silently vanish on restart (no
  compile error).
- The feature intentionally makes commit diff **reviewable** (`review_diff`) and
  **splittable** (`clone_on_split`) and **persistent** — these are easy to forget
  because they're refutable-pattern sites, not exhaustive matches.

---

## 9. Current state (as of writing)

- Branch `mg/git_branch_diff_any`, feature commit `git diff commit view`
  (SHA `bc96646f16` at write time — will change).
- **The branch is currently ~903 commits behind `main`** (merge-base `4ebc1545d2`,
  main `b54cc1d0ac`). i.e. the next task is almost certainly *another rebase* per §6.
- The commit touches exactly 4 files:
  - `crates/git_ui/src/branch_diff.rs` (bulk of feature; +~303/-…)
  - `crates/git_ui/src/git_ui.rs` (RefPickerModal + RefPickerAction; +69)
  - `crates/git_ui/src/project_diff.rs` (CommitDiff action decl; +3)
  - `crates/project/src/git_store/diff_buffer_list.rs` (DiffBase::Commit + matches; +43)
- Working tree clean. Backups from prior rebases exist as `backup-mg-git_branch_diff_any-*`.
- Not pushed. Upstream diverges after rebase (expected).

## 10. Reference: essence of the key additions

These are the *intent* snippets to re-apply (adapt names/paths to current `main`).

`DiffBase` (diff_buffer_list.rs):
```rust
pub enum DiffBase {
    Head, Index, Staged,
    Merge  { base_ref: SharedString },   // branch diff
    Commit { base_ref: SharedString },   // THIS FEATURE: arbitrary ref, same merge-base engine
}
impl DiffBase {
    pub fn requires_tree_diff(&self) -> bool {
        matches!(self, DiffBase::Merge { .. } | DiffBase::Commit { .. })
    }
    pub fn base_ref(&self) -> Option<&SharedString> {
        match self {
            DiffBase::Head | DiffBase::Index | DiffBase::Staged => None,
            DiffBase::Merge { base_ref } | DiffBase::Commit { base_ref } => Some(base_ref),
        }
    }
}
```

`deploy_commit_diff` (branch_diff.rs) — reuse-or-create a single commit-diff tab:
```rust
pub(crate) fn deploy_commit_diff(workspace, base_ref: SharedString, window, cx) {
    telemetry::event!("Git Commit Diff Opened");
    let project = workspace.project().clone();
    let intended_repo = project.read(cx).active_repository(cx);
    if let Some(existing) = workspace.items_of_type::<Self>(cx)
        .find(|it| matches!(it.read(cx).diff_base(cx), DiffBase::Commit { .. })) {
        workspace.activate_item(&existing, true, true, window, cx);
        existing.update(cx, |cd, cx| {
            if let Some(r) = intended_repo { cd.set_repo(Some(r), cx); }
            cd.set_commit_base(base_ref, cx);
        });
        return;
    }
    let ws = cx.entity();
    let cd = cx.new(|cx| Self::new_with_diff_base(
        DiffBase::Commit { base_ref }, project, ws, intended_repo, window, cx));
    workspace.add_item_to_active_pane(Box::new(cd), None, true, window, cx);
}
```

Toolbar base selector (BranchDiffToolbar::render):
```rust
let base_selector: AnyElement = match &diff_base {
    DiffBase::Commit { .. } => Button::new("commit-diff-base-ref", base_ref_label)
        .end_icon(Icon::new(IconName::ChevronDown).size(IconSize::XSmall).color(Color::Muted))
        .tooltip(Tooltip::text("Change Ref"))
        .on_click(|_, window, cx| window.dispatch_action(CommitDiff.boxed_clone(), cx))
        .into_any_element(),
    DiffBase::Merge { .. } => /* existing PopoverMenu branch picker */ .into_any_element(),
    DiffBase::Head | DiffBase::Index | DiffBase::Staged => return div(),
};
```

git_ui.rs confirm handler:
```rust
RefPickerAction::CommitDiff => {
    crate::branch_diff::BranchDiff::deploy_commit_diff(
        workspace, git_ref_string.clone().into(), window, cx);
}
```
