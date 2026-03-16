# LoopWorkspace Patch Creation Guide

## Common Pitfalls and Best Practices

Based on experience creating patches for the Loop Apple Watch app, here are the critical lessons learned.

---

## 1. File Path Structure (CRITICAL)

**The #1 mistake: Incorrect file paths**

All paths in patches must include the `Loop/` submodule directory prefix.

- ❌ **WRONG**: `Common/Models/WatchContext.swift`
- ✅ **CORRECT**: `Loop/Common/Models/WatchContext.swift`

**Why**: The GitHub Actions workflow runs `git apply ./patches/*` from the LoopWorkspace root directory. The actual source code lives inside the `Loop/` git submodule, not in the workspace root.

---

## 2. Correct Patch Creation Workflow

**The #2 mistake: Creating patches manually**

Never write patches by hand with fake commit hashes.

### Proper Method:

**CRITICAL**: Generate patches from the **workspace root**, NOT from within the submodule.

```bash
# 1. Ensure submodules are initialized
cd LoopWorkspace
git submodule update --init --recursive

# 2. Enter the Loop submodule and make changes
cd Loop
# ... edit files in Loop/WatchApp Extension/, Loop/Loop/, etc. ...

# 3. Return to workspace root and generate patch from there
cd ..
git diff Loop/ > patches/descriptive-name.patch

# 4. Verify paths are correct (should start with Loop/)
head -5 patches/descriptive-name.patch

# 5. Commit and push
git add patches/descriptive-name.patch
git commit -m "Add: description of your change"
git push origin main
```

**Why this works**: 
- `git diff Loop/` generates proper unified diff format with real commit hashes
- Paths include `Loop/` prefix, which is required for CI/CD to find the files
- CI/CD runs `git apply` from workspace root, so paths must be workspace-relative

**⚠️ WARNING**: If you generate from within the submodule (`cd Loop && git diff`), paths will be missing the `Loop/` prefix and CI/CD will fail with "No such file or directory".

---

## 3. Understand the Repository Structure

```
LoopWorkspace/                    ← Root (workspace)
├── patches/                      ← Your patches go here
│   └── your-patch.patch          ← Will be applied during build
├── Loop/                         ← Git submodule (main app code)
│   ├── Common/                   ← Shared models
│   ├── Loop/                     ← iPhone app code
│   ├── WatchApp Extension/       ← Watch app code
│   └── ...
├── LoopKit/                      ← Another submodule
├── CGMBLEKit/                    ← Another submodule
└── ...
```

**Key Insight**: Most changes for the Watch app are in `Loop/WatchApp Extension/`, not in the workspace root.

---

## 4. Initialize Submodules First

**Before making any changes**:

```bash
git submodule update --init --recursive
```

**Verify it worked**:
```bash
ls Loop/  # Should show directories, not be empty
```

If submodules are empty, the patch generation will fail or create incorrect paths.

---

## 5. How Patches Are Applied in CI/CD

From `.github/workflows/build_loop.yml`:

```yaml
# Customize Loop: Download and apply patches
- name: Customize Loop
  run: |
    # LoopWorkspace patches
    if $(ls ./patches/* &> /dev/null); then
      git apply ./patches/* --allow-empty -v --whitespace=fix
    fi
```

**Important**: This runs from the workspace root, which is why paths need the `Loop/` prefix.

---

## 6. Testing Patches Locally

**Before committing**, verify the patch applies cleanly:

```bash
cd LoopWorkspace/Loop
git apply --check ../patches/your-patch.patch
```

If this shows errors, fix them before pushing.

**Common errors**:
- `No such file or directory` → Missing `Loop/` prefix in paths
- `corrupt patch at line X` → Malformed patch format
- `patch does not apply` → Conflicts with current code

---

## 7. Battery and Performance Considerations

When adding features to the Watch app:

**✅ GOOD**:
- One extra data fetch per update cycle (every 5-15 minutes)
- Simple database queries (TDD calculation)
- Small data transfer (one extra Double value)

**❌ BAD**:
- Continuous background polling
- Large data transfers
- Complex calculations on the Watch

**Rule of thumb**: Keep changes minimal. The heavy lifting should happen on the iPhone, not the Watch.

---

## 8. Handling Upstream Updates

When LoopKit releases updates:

1. The sync workflow pulls upstream changes
2. Your patches are applied on top
3. If conflicts occur, the build fails

**Solution**: Regenerate the patch from the updated upstream code:
```bash
cd Loop
git fetch upstream
git merge upstream/main
# Re-apply your changes manually
git diff > ../patches/your-patch.patch
```

**Tip**: Keep patches small and focused to reduce conflict risk.

---

## 9. Localization

If adding UI text that needs translation:

1. Add the string to `Loop/WatchApp Extension/Localizable.xcstrings`
2. Include at least English localization
3. German (de) is helpful for EU users

**Example**:
```json
"TDD" : {
  "comment" : "HUD row title for TDD",
  "localizations" : {
    "de" : { "stringUnit" : { "value" : "TDD" } },
    "en" : { "stringUnit" : { "value" : "TDD" } }
  }
}
```

---

## 10. Quick Reference: Creating a Patch

```bash
# 1. Ensure submodules are initialized
cd LoopWorkspace
git submodule update --init --recursive

# 2. Enter submodule and make changes
cd Loop
# ... edit files ...

# 3. Generate patch
git diff > ../patches/feature-name.patch

# 4. Test patch applies
git apply --check ../patches/feature-name.patch

# 5. Commit and push
cd ..
git add patches/feature-name.patch
git commit -m "Add: feature description"
git push origin main

# 6. Trigger build (manual or wait for scheduled)
# Go to GitHub Actions → "4. Build Loop" → Run workflow
```

---

## Summary Checklist

Before pushing a patch:

- [ ] Submodules initialized (`git submodule update --init --recursive`)
- [ ] Changes made in `Loop/` directory, not workspace root
- [ ] Patch generated with `git diff > ../patches/name.patch`
- [ ] Paths include `Loop/` prefix
- [ ] Patch applies cleanly (`git apply --check`)
- [ ] Working tree clean (`git status`)
- [ ] Committed and pushed
- [ ] Build triggered or scheduled

---

## Troubleshooting

| Error | Cause | Solution |
|-------|-------|----------|
| `No such file or directory` | Missing `Loop/` prefix | Add `Loop/` to all paths |
| `corrupt patch at line X` | Malformed patch | Regenerate with `git diff` |
| `patch does not apply` | Conflicts or wrong base | Update to latest upstream, regenerate |
| `trailing whitespace` | Whitespace issues | Use `--whitespace=fix` flag |
| Files not found | Submodules not initialized | Run `git submodule update --init --recursive` |

---

## Example: TDD Toggle Patch Structure

This patch adds a toggle between IOB and TDD on the Apple Watch:

**Files modified**:
1. `Loop/Common/Models/WatchContext.swift` - Add TDD property
2. `Loop/WatchApp Extension/Extensions/WatchContext+WatchApp.swift` - Add computed property
3. `Loop/Loop/Managers/WatchDataManager.swift` - Fetch TDD data
4. `Loop/WatchApp Extension/Controllers/ChartHUDController.swift` - Add toggle logic
5. `Loop/WatchApp Extension/Controllers/HUDRowController.swift` - Add display method
6. `Loop/WatchApp Extension/Localizable.xcstrings` - Add translations

**Key learning**: All 6 files have `Loop/` prefix in the patch paths.

---

*Document created based on experience adding TDD toggle feature to Loop Apple Watch app.*
