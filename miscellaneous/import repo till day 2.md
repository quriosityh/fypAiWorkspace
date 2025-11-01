---
share_link: https://share.note.sx/vul2bhi2#dlT+/EVMuD9K2Hm3mn5iwVHFqDmelfsOncr1lBP14dw
share_updated: 2025-11-01T17:54:02+05:00
---

Perfect 👌—you’ve done the hard part! Since the full **StuFlux monorepo** is already set up, committed, and pushed to GitHub (with `main` and `develop` branches + collaborator access), your teammate just needs to **clone and bootstrap** the project on their local system to match your setup exactly.

Here’s the **complete step-by-step setup guide** for your teammate 👇

---

# 🧑‍💻 Teammate Setup Guide (Mirror StuFlux Local Setup)

## ⚙️ Prerequisites (must install first)

Before cloning the repo, make sure your teammate has these installed globally:

| Tool        | Minimum Version                          | Command to Verify |
| ----------- | ---------------------------------------- | ----------------- |
| **Node.js** | ≥ 18.0.0                                 | `node -v`         |
| **pnpm**    | ≥ 8.15.0 (you’re on 10.20.0, compatible) | `pnpm -v`         |
| **Git**     | ≥ 2.34                                   | `git --version`   |

> 💡 On Linux Mint / Ubuntu:
>
> ```bash
> sudo apt update
> sudo apt install -y nodejs npm git
> npm install -g pnpm@latest
> ```

---

## 🧩 Step 1: Clone the Repository

```bash
# Go to your preferred development folder
cd ~/codeSPACE  # or wherever you keep your projects

# Clone StuFlux repo
git clone https://github.com/<your-username>/stuflux.git

cd stuflux
```

---

## 🌿 Step 2: Setup Git Branches

```bash
# Fetch all branches
git fetch --all

# Switch to the develop branch (main for development)
git checkout develop
```

> ✅ `main` → production
> ✅ `develop` → working base branch
> ✅ `feature/*` → for individual feature work

---

## 📦 Step 3: Install All Dependencies (Monorepo-wide)

Since this is a **pnpm workspace**, she should install **once** at the root:

```bash
# At project root (stuflux/)
pnpm install
```

🧠 This will automatically install dependencies for:

* `apps/web`
* `apps/api`
* `packages/types`

…and symlink them together properly via the workspace.

---

## ⚙️ Step 4: Verify Folder Structure

After install, she should see this structure:

```
stuflux/
├── apps/
│   ├── web/        # Next.js frontend
│   └── api/        # Express backend
├── packages/
│   └── types/      # Shared TypeScript types
├── pnpm-workspace.yaml
├── package.json
└── ...
```

If something looks missing (like `node_modules` in subfolders), she can re-run:

```bash
pnpm install --recursive
```

---

## 🧾 Step 5: Environment Files

She must create her own `.env` files by **copying** the provided examples.

### For Frontend (`apps/web`):

```bash
cd apps/web
cp .env.example .env.local
```

Then edit `.env.local` and replace with valid credentials:

* Database URL (from Neon)
* Clerk keys
* Cloudinary keys

### For Backend (`apps/api`):

```bash
cd ../api
cp .env.example .env
```

Then fill in values for:

* `PORT`, `DATABASE_URL`, Clerk and Cloudinary keys
* Make sure `ALLOWED_ORIGINS` includes `http://localhost:3000`

---

## 🚀 Step 6: Run Both Apps in Dev Mode

From the root folder (`stuflux/`):

```bash
pnpm dev
```

✅ This will:

* Start **Next.js (frontend)** on `http://localhost:3000`
* Start **Express (backend)** on `http://localhost:4000`

> 🧠 Under the hood:
>
> ```bash
> concurrently "pnpm -F web dev" "pnpm -F api dev"
> ```

---

## 🧪 Step 7: Test the Setup

### Test Backend:

Visit:

```
http://localhost:4000/api/v1/health
```

Expected response:

```json
{
  "status": "ok",
  "timestamp": "...",
  "version": "1.0.0"
}
```

### Test Frontend:

Visit:

```
http://localhost:3000
```

Next.js should load successfully.

---

## 🧼 Step 8: Verify Type Safety & Linting

Run these checks to ensure her environment matches yours:

```bash
# TypeScript check (entire monorepo)
pnpm type-check

# Lint all code
pnpm lint
```

---

## 🧰 Step 9: Branch Workflow (Team Sync Rules)

When contributing:

```bash
# Always update develop before starting new work
git checkout develop
git pull origin develop

# Create feature branch
git checkout -b feature/<short-feature-name>

# Push it
git push -u origin feature/<short-feature-name>
```

After development, open a **Pull Request → develop** branch.
Merges to **main** should only happen through reviewed PRs.

---

## 🪄 Optional: Recommended VS Code Setup

To match your environment:

1. Install extensions:

   * **ESLint**
   * **Prettier**
   * **Tailwind CSS IntelliSense**
   * **TypeScript Hero**
2. Enable format on save:

   ```json
   "editor.formatOnSave": true,
   "editor.defaultFormatter": "esbenp.prettier-vscode"
   ```

---

## 🧩 Step 10: Common Troubleshooting

| Issue                       | Fix                                                         |
| --------------------------- | ----------------------------------------------------------- |
| `pnpm: command not found`   | Install pnpm globally: `npm install -g pnpm`                |
| `Port already in use`       | Change port in `.env` or kill process: `sudo lsof -i :4000` |
| Next.js “invalid env var”   | Ensure `.env.local` is in `/apps/web` not root              |
| Type errors after clone     | Run `pnpm rebuild` or `pnpm install --force`                |
| Express server not starting | Check `.env` → correct `PORT` and `DATABASE_URL`            |

---

## ✅ Final Checklist for Teammate

* [ ] Installed Node 18+, pnpm 8.15+
* [ ] Cloned repo and checked out `develop`
* [ ] Ran `pnpm install` at root
* [ ] Created `.env` files for both apps
* [ ] Confirmed both servers run with `pnpm dev`
* [ ] Verified health check endpoint works
* [ ] Synced Git branch workflow (`feature/*`, PRs to `develop`)

---

Would you like me to generate a **“StuFlux_Teammate_Setup.md”** file that you can directly add to your repo (`docs/`) so she can follow it step-by-step from there?
