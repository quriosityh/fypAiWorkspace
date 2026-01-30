# Development Guide

## Database Access with Drizzle ORM

We use Drizzle ORM for database operations. See [[lessons/drizzle-setup]] for detailed setup and usage guide.

Key points:
- Schema is defined in `apps/api/src/db/schema.ts`
- Use Drizzle's type-safe query builders instead of raw SQL
- Run migrations with `pnpm db:generate` and `pnpm db:push`
- Use the Drizzle Studio UI with `pnpm db:studio` for data exploration

## Git Workflow

Here's a **visual explanation of the Git branching workflow** for `main`, `develop`, and `feature/*` branches in a monorepo like your StuFlux project:

```
          ┌─────────────────────────┐
          │        main             │  ← Protected production-ready branch
          └──────────┬──────────────┘
                     │
                     │ Merge PRs only
                     │
          ┌──────────▼──────────────┐
          │       develop           │  ← Active development branch
          └──────────┬──────────────┘
                     │
       ┌─────────────┴─────────────┐
       │                           │
       ▼                           ▼
feature/login-system      feature/payment-integration
       │                           │
       │ Code / commits             │ Code / commits
       │                           │
       └─────────────┬─────────────┘
                     Merge PRs back into develop
```

### How it works step by step:

1. **`main` branch**

   * Always stable, production-ready.
   * Protected: changes can only come via **pull requests** (PRs) with reviews.

2. **`develop` branch**

   * Base branch for all new features.
   * All feature branches branch off from `develop`.
   * After testing, `develop` merges into `main` for release.

3. **Feature branches (`feature/*`)**

   * Named like `feature/login-system`, `feature/payment-integration`.
   * Work happens here.
   * Once ready, submit a PR into `develop`.
   * Keeps features **isolated** from each other.

4. **Pull Request Workflow**

   * Developer creates a feature branch.
   * Makes commits & pushes to remote.
   * Opens PR into `develop`.
   * Code review happens.
   * Merge PR → develop.

5. **Release**

   * Once `develop` has stable features, open a PR to merge into `main`.
   * Production deploys from `main`.

---

This workflow is called **Git Flow** — very common in team projects.

If you want, I can also **draw the same diagram showing multiple developers contributing simultaneously**, so you can see how conflicts and merges work in a monorepo.

Do you want me to do that too?


# Commands for  workflow
## 0️⃣ One-time setup (do this once)

```bash
git config --global user.name "Your Name"
git config --global user.email "you@email.com"
```

Optional but clutch:

```bash
git config --global pull.rebase true
```

👉 keeps history clean, avoids merge spaghetti.

---

## 1️⃣ Cloning the repo (first day on project)

```bash
git clone <repo-url>
cd stuflux
```

Check branches:

```bash
git branch -a
```

Switch to develop:

```bash
git checkout develop
```

Or modern way:

```bash
git switch develop
```

---

## 2️⃣ Starting new work (MOST IMPORTANT PART)

⚠️ **Rule:**

> NEVER work directly on `main` or `develop`

### Create a feature branch

```bash
git switch develop
git pull origin develop
git switch -c feature/login-system
```

What this does:

- `switch develop` → base branch
    
- `pull` → latest code
    
- `-c` → create new branch
    

🔥 Now you’re safe to code.

---

## 3️⃣ Day-to-day coding loop (you’ll repeat this nonstop)

### Check status

```bash
git status
```

### See changes

```bash
git diff
```

### Stage files

```bash
git add .
```

or specific:

```bash
git add apps/api/src/listings/service.ts
```

### Commit

```bash
git commit -m "feat: add listing filters query"
```

🔥 Commit message rule (follow this):

- `feat:` new feature
    
- `fix:` bug fix
    
- `refactor:` code cleanup
    
- `chore:` config / tooling
    

---

## ==4️⃣ Push your feature branch==

```bash
git push -u origin feature/login-system
```

`-u` = remembers upstream, so next time:

```bash
git push
```

---

## 5️⃣ Opening a PR (mental model)

You don’t do this in terminal usually, but logically:

```
feature/login-system  →  develop
```

After PR:

- CI runs
    
- Code review
    
- Merge happens
    

⚠️ You do NOT merge locally unless your team allows it.

---

## 6️⃣ Syncing your branch with latest develop (VERY IMPORTANT)

Develop moves fast. You MUST keep up.

### While on your feature branch:

```bash
git fetch origin
git rebase origin/develop
```

If conflicts happen:

```bash
# fix files
git add .
git rebase --continue
```

Why rebase?

- Clean history
    
- No “merge commit hell”
    

---

## 7️⃣ When PR is merged → clean up

After merge is done on GitHub:

```bash
git switch develop
git pull origin develop
git branch -d feature/login-system
```

Remote cleanup (optional):

```bash
git push origin --delete feature/login-system
```

---

## 8️⃣ Release flow (develop → main)

Usually done by lead/devops, but you should understand it:

```bash
git switch develop
git pull origin develop
git switch main
git pull origin main
```

Then PR:

```
develop → main
```

Deploy from `main`. Period.

---

## 9️⃣ Emergency commands (you WILL need these)

### Undo last commit (keep code)

```bash
git reset --soft HEAD~1
```

### Discard local changes

```bash
git checkout -- .
```

### Stash work (pause feature)

```bash
git stash
git switch develop
```

Come back later:

```bash
git switch feature/xyz
git stash pop
```

---

## 10️⃣ Visualize what’s going on (underrated)

```bash
git log --oneline --graph --all
```

This will make Git finally make sense.

---

## 🧠 Mental Rules (tattoo these)

❌ Don’t commit broken code  
❌ Don’t push to `main`  
❌ Don’t mix features in one branch  
✅ Small commits  
✅ Rebase often  
✅ One feature = one branch

---

## What you’re basically doing

```text
main      → sacred
develop   → playground
feature/* → chaos (but isolated)
```

---

If you want next, I can:

- Simulate **2 devs causing a merge conflict**
    
- Teach **rebase vs merge** with visuals
    
- Show **Git mistakes that nuke repos** (so you never do them)
    

Say what you want next.