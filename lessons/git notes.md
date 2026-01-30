## git clean vs git restore 

* **`git restore` → undo changes to *tracked* files**
* **`git clean` → delete *untracked* files**

That’s it. That’s the line.

---

## Side-by-side truth table

| Situation                    | `git restore` | `git clean` |
| ---------------------------- | ------------- | ----------- |
| Undo edits in a tracked file | ✅ YES         | ❌ NO        |
| Unstage a file               | ✅ YES         | ❌ NO        |
| Delete untracked files       | ❌ NO          | ✅ YES       |
| Remove `node_modules/`       | ❌ NO          | ✅ YES       |
| Safe by default              | ⚠️ Medium     | ❌ Nope      |
| Touches committed history    | ❌ NO          | ❌ NO        |

---

## `git restore` (undo, not delete)

Use this when:

* You edited a file and regret it
* You want the last committed version back
* You staged something by mistake

### Examples

```bash
git restore app.js        # discard edits
git restore .             # discard all local edits
git restore --staged app.js  # unstage but keep edits
```

👉 Works **only on tracked files**
👉 Does **not** delete files

---

## `git clean` (delete, no mercy)

Use this when:

* Repo is messy
* You want a fresh slate
* `node_modules`, `dist`, temp junk everywhere

### Examples

```bash
git clean -nfd   # preview (SAFE)
git clean -fd    # DELETE untracked files (DANGEROUS)
```

👉 Deletes files forever
👉 Does **not** touch tracked files

---

## Real-world analogy 🧠

* `git restore` = “Undo my edits”
* `git clean` = “Take out the trash”

Don’t use a trash can to undo edits. Don’t use undo to remove trash.

---

## Absolute golden rule 🏆

**Always run:**

```bash
git clean -nfd
```

before:

```bash
git clean -fd
```

---

## TL;DR

* Edited tracked files → `git restore`
* Junk untracked files → `git clean`
* Want to lose work fast → misuse either 😈

If you want, tell me **what your repo looks like right now** (edited files? staged? junk dirs?) and I’ll tell you **exactly which command to run, safely**.
