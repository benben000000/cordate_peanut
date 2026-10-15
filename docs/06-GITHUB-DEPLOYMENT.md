# 06 — Git and GitHub Repository Deployment Setup

Target remote: `https://github.com/benben000000/cordate_peanut.git` · Git user: `benben000000`
· Email: `bmgarcia0121@gmail.com` · Primary branch: `main`

Remote state verified 2026-10-15: repository exists, is public, and has **no branches** (empty).
All command blocks below are copy-pasteable (bash; PowerShell equivalents noted where they differ).

---

## 6.0 Prerequisite: Git runtime

This build environment had no `git` on PATH, so a user-scoped portable install was used
(no admin rights, no machine-wide changes):

```powershell
# Windows, no admin: portable Git 2.56.0.2, hash-verified before extraction
$url  = 'https://github.com/git-for-windows/git/releases/download/v2.56.0.windows.2/PortableGit-2.56.0.2-64-bit.7z.exe'
$dest = "$env:LOCALAPPDATA\PortableGit"
Invoke-WebRequest -Uri $url -OutFile "$env:TEMP\PortableGit.7z.exe" -UseBasicParsing
# SHA-256 must equal 075e158ef8e1f0ab80b347e245405d3eca735c2dc88fd8e032e137d0ca61f61b
& "$env:TEMP\PortableGit.7z.exe" -o"$dest" -y
& "$dest\cmd\git.exe" --version          # git version 2.56.0.windows.2
$env:Path = "$dest\cmd;$env:Path"        # current session
```

Other platforms: `winget install Git.Git` (Windows with winget), `brew install git` (macOS),
`sudo apt install git` (Debian/Ubuntu). No `gh` CLI is required; all steps use plain `git`.
`gh` is only an optional convenience (§6.5).

## 6.1 Step 1 — Identity configuration

```bash
cd /path/to/cordate_peanut

# Repository-local identity (does not touch global config)
git config --local user.name  "benben000000"
git config --local user.email "bmgarcia0121@gmail.com"

# Deterministic line endings on Windows (repo stores LF, per .gitattributes)
git config --local core.autocrlf false
git config --local core.eol lf

# Verify
git config --local --list
```

Expected output includes `user.name=benben000000`, `user.email=bmgarcia0121@gmail.com`,
`core.autocrlf=false`, `core.eol=lf`.

## 6.2 Step 2 — Repository initialization

```bash
git init -b main
git add .gitignore .gitattributes
git commit -m "chore: initialize repository with ignore rules and LF policy"
```

If an old Git (< 2.28) rejects `-b main`: `git init && git checkout -b main`.

## 6.3 Step 3 — Stage documentation and project structure

The suite generated for this repo:

```
README.md
NOTICE.md
.gitignore
.gitattributes
docs/01-PRD.md
docs/02-TRD.md
docs/03-SYSTEM-FLOW.md
docs/04-IMPLEMENTATION-PLAN.md
docs/05-UIUX-SPEC.md
docs/06-GITHUB-DEPLOYMENT.md
docs/07-FREE-TIER-LEDGER.md
```

```bash
# Secrets guard before staging (must print nothing)
git check-ignore -v .env .env.local secrets/firebase-admin.json 2>/dev/null
grep -rnE "(AIza[0-9A-Za-z_-]{35}|-----BEGIN [A-Z ]*PRIVATE KEY-----)" --include="*.md" --include="*.json" . \
  && echo "STOP: secret found" || echo "clean"

git add docs/ README.md NOTICE.md
git status --short
```

## 6.4 Step 4 — Conventional commit sequence

Commit in dependency order so each commit is reviewable in isolation:

```bash
# 4.1 Product and architecture specifications
git add docs/01-PRD.md docs/02-TRD.md docs/03-SYSTEM-FLOW.md
git commit -m "docs: add invisible pedagogy system architecture and GCP specs" \
           -m "PRD, TRD (schemas, endpoints, quota profiles), system flow with fallback ladder."

# 4.2 Planning and interface contracts
git add docs/04-IMPLEMENTATION-PLAN.md docs/05-UIUX-SPEC.md
git commit -m "docs: add 48-hour implementation plan and ambient UI/UX specification" \
           -m "Phase gates with measurable acceptance criteria, cue catalog, zero-friction error states."

# 4.3 Deployment and free-tier compliance
git add docs/06-GITHUB-DEPLOYMENT.md docs/07-FREE-TIER-LEDGER.md
git commit -m "docs: add deployment workflow and free-tier quota ledger" \
           -m "Verified quota math for Firebase Spark, Cloud Run grants, BigQuery sandbox, Gemini free tier."

# 4.4 Repository entry points
git add README.md NOTICE.md
git commit -m "docs: add repository index and CC-BY-4.0 attribution notice" \
           -m "Quickstart for zero-card profile, model registry, paper attribution (DOI 10.5281/zenodo.21052875)."

git log --oneline --decorate
```

## 6.5 Step 5 — Remote binding and push

```bash
git remote add origin https://github.com/benben000000/cordate_peanut.git
git remote -v
git branch -M main
git push -u origin main
```

Authentication options when the push prompts (HTTPS requires credentials; none were present in
this environment):

| Option | Commands | Notes |
| --- | --- | --- |
| PAT (quickest) | `git push https://benben000000:<TOKEN>@github.com/benben000000/cordate_peanut.git main` | Token needs `repo` scope (classic) or Contents: read/write (fine-grained). Never store the token in the remote URL on disk; use the one-shot form above and then `git remote set-url origin https://github.com/benben000000/cordate_peanut.git` |
| Credential helper (persistent) | `git config --local credential.helper manager` then `git push -u origin main` | Opens the browser device flow on desktop Git installs; portable Git includes no helper — install the full Git for Windows or `git-credential-manager` first |
| SSH | `ssh-keygen -t ed25519 -C "bmgarcia0121@gmail.com"` → add the public key to GitHub → `git remote set-url origin git@github.com:benben000000/cordate_peanut.git` → `git push -u origin main` | Requires an SSH client on the machine |
| GitHub CLI (optional) | `gh auth login` → `gh repo create benben000000/cordate_peanut --public --source=. --push` | Not installed in this environment |

## 6.6 Step 6 — Post-push housekeeping

```bash
# Verify what the remote actually received
git ls-remote --heads origin

# Release tag for the submission freeze (Gate G4)
git tag -a v1.0.0 -m "Invisible Pedagogy AI — contest submission freeze"
git push origin v1.0.0

# Optional: keep the demo video out of Git; attach to a GitHub Release if > 100 MB
# (GitHub hard-limits files at 100 MB; repository soft-guidance 1 GB)
```

Branch hygiene during the sprint: `feat/telemetry`, `feat/gemini-pipeline`, `feat/ambient-ui`,
`feat/analytics`; merge to `main` at gates only; `main` is the only branch judges build from.

## 6.7 Judge quickstart path (referenced by the submission checklist)

```bash
git clone https://github.com/benben000000/cordate_peanut.git && cd cordate_peanut
cp .env.example .env.local     # repo ships demo defaults that work without Firebase (local-only mode)
pnpm install && pnpm dev       # http://localhost:3000
# keyboard: Ctrl+Shift+D toggles the Cognitive State Debugger
# keyboard: Ctrl+Shift+F runs the scripted friction scenarios (backspace storm, idle decay, undo loop)
```

Local-only mode requires no accounts: RTDB/Firestore listeners are disabled, the gateway is optional
(static demo data), and all cues originate from the on-device classifier.

## 6.8 Troubleshooting

| Symptom | Cause | Fix |
| --- | --- | --- |
| `fatal: could not read Username` | no credentials available | use an auth option from §6.5 |
| `remote: Support for password authentication was removed` | plain password used | use a PAT or SSH |
| `fatal: refusing to merge unrelated histories` | remote has commits (it does not today) | `git pull origin main --allow-unrelated-histories` |
| CRLF churn in `git status` | `core.autocrlf` left at `true` | `git config --local core.autocrlf false && git add --renormalize .` |
| 403 on push to fine-grained token | missing Contents: write | re-scope the token |
| Accidentally committed a secret | push already done | rotate the credential first, then `git rm --cached <file>`, commit, push; history rewrite (`git filter-repo`) only if the repo is public and the secret is live |
