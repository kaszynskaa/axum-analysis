# Fix the Bug

## Bugs found and fixed

### 1. Empty stray directory `docs/docs2`
**Problem:** An empty `docs2/` directory existed inside `docs/`, created accidentally during navigation.  
**Fix:** Removed with `rmdir docs/docs2`.

---

### 2. Missing `.gitignore` (root cause of bugs 3 & 4)
**Problem:** No `.gitignore` existed, so macOS system files and other unwanted files were being picked up by git.  
**Fix:** Created `.gitignore` at the project root:
```
.DS_Store
**/.DS_Store
```

---

### 3. `.DS_Store` files committed to git
**Problem:** Two macOS metadata files were tracked by git:
- `.DS_Store`
- `docs/.DS_Store`

**Fix:** Removed from git tracking with `git rm --cached`.

---

### 4. Typo in filename `previw-diagrams.png`
**Problem:** File was named `previw-diagrams.png` (missing `e`).  
**Fix:** Renamed to `preview-diagrams.png` with `git mv`.

---

## Commits

| Hash | Message |
|---|---|
| `a049029` | Remove empty docs/docs2 directory |
| `ea85b2d` | Fix gitignore, remove .DS_Store, fix typo in PNG filename |
