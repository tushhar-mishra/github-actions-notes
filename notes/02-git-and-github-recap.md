# 02 · Git and GitHub Recap (Everything GitHub Actions Needs)

> **Part A: Foundations** · Previous: [01 · What is CI/CD?](01-what-is-ci-cd.md) · Next: [03 · YAML for GitHub Actions](03-yaml-for-github-actions.md) · [Back to INDEX](INDEX.md)
>
> Every lab in these notes starts with "set up the repo and authenticate". That setup lives **only here**. Later labs say "do the setup from topic 02".

---

## What it is

- **Git** is a tool on your laptop that records every version of your code. Each saved version is a **commit**.
- **GitHub** is a website that hosts Git repositories online so teams can share them.
- A **repository** (**repo**) is a project folder that Git tracks. It has a hidden `.git/` folder that stores all history.
- A **branch** is a separate line of work. `main` is the main line. You make a `feature/...` branch to try changes safely.
- A **remote** is the online copy of your repo. By convention it's called `origin`.
- A **push** sends your commits from your laptop to the remote.
- A **pull request** (**PR**) is a request on GitHub that says "please review my branch and merge it into `main`".

GitHub Actions lives **inside** a GitHub repo. It reacts to Git things: **pushes, branches, tags, pull requests**.

---

## Why it exists

GitHub Actions only knows that something happened because of Git activity on GitHub. If you can't create a repo, commit, push a branch and open a PR, you can't trigger or test a single workflow.

**Jenkins comparison:** Jenkins sits **outside** GitHub. To react to a push it needs a **webhook** (an HTTP message GitHub sends to Jenkins saying "something happened"), plus credentials so it can clone the repo. GitHub Actions is built into GitHub, so the push itself is the trigger. No webhook, no extra credentials.

---

## Analogy

- **Git** is a **notebook with infinite undo**. Every commit is a dated page you can go back to.
- A **branch** is a **photocopy of the notebook** where you can scribble without ruining the original.
- **GitHub** is the **shared library shelf** where the team keeps the official copy.
- A **push** is **handing your new pages to the librarian**.
- A **pull request** is **asking the editor to review your photocopy** before it's added to the official copy.
- **GitHub Actions** is a **robot assistant at the library**. Whenever pages arrive (a push) or a review is requested (a PR), it reads them, checks them, and reports back.

---

## How it works inside

### How SSH authentication works

Your laptop must prove to GitHub that it's really you before GitHub accepts a push. There are three common ways:

- **HTTPS + password**: no longer allowed for Git operations on GitHub.
- **HTTPS + PAT** (Personal Access Token: a long random password you generate on GitHub, with limited permissions and an expiry).
- **SSH** (Secure Shell): a **key pair**. This course uses SSH because it's the most common approach in companies.

How SSH works:

1. You generate **two keys** that are mathematically linked: a **private key** (secret, stays on your laptop) and a **public key** (safe to share).
2. You give GitHub the **public key**.
3. When you push, GitHub sends a challenge. Your laptop answers it using the **private key**.
4. GitHub checks the answer with the public key. If it matches, you're in. The private key never leaves your machine.

### What happens when you push

```text
your laptop                       GitHub
-----------                       ------
git commit  → saves a snapshot locally (.git/)
git push    → sends new commits ─────────►  remote branch updated
                                            │
                                            ▼
                                  GitHub records a "push" event
                                            │
                                            ▼
                                  GitHub Actions checks .github/workflows/
                                  for workflows that listen to "push"
                                  (the full story is in topic 08)
```

---

## Workflow YAML / commands

### Step 1: Create a private repository on GitHub

Most company apps live in **private** repos, so this course uses one too. It also makes GitHub Actions behave the way it does in real organizations.

Create a repo with these details:

| Property | Value |
|---|---|
| Repository name | `cwvj-gha-practice` (any name works) |
| Visibility | **Private** |

Do **not** tick "initialize with README", `.gitignore`, or license. You'll push everything from your laptop.

> Some later labs use other repo names (for example `cwvj-flask-app`). The setup is identical.

### Step 2: Create an SSH key pair

```bash
ssh-keygen -t ed25519 -C "your-email@example.com" -f ~/.ssh/cwvj_gha_ed25519
```

| Part | Meaning |
|---|---|
| `ssh-keygen` | The program that creates SSH keys. |
| `-t ed25519` | The key **type**. Ed25519 is modern, secure and recommended. |
| `-C "..."` | A **comment** (a label) stored in the key. People use their email so GitHub can link the key to an account. It is **not** used for authentication, it's only for identification, and it doesn't have to be an email. |
| `-f ~/.ssh/cwvj_gha_ed25519` | The **file name and location**. Important when you keep several keys. It creates `cwvj_gha_ed25519` (private) and `cwvj_gha_ed25519.pub` (public). |

**Sample output:**

```text
Generating public/private ed25519 key pair.
Enter passphrase (empty for no passphrase):
Your identification has been saved in /home/you/.ssh/cwvj_gha_ed25519
Your public key has been saved in /home/you/.ssh/cwvj_gha_ed25519.pub
The key fingerprint is:
SHA256:3vX1...q9 your-email@example.com
```

- The **private key** stays on your machine. **Never share it.**
- The **public key** goes to GitHub so GitHub can recognize your machine.

### Step 3: Load the key into the SSH agent

The **SSH agent** is a small background program that keeps your keys in memory for the current session, so Git doesn't ask for them on every push.

```bash
# Start the SSH agent
eval "$(ssh-agent -s)"

# Add the private key to the SSH agent
ssh-add ~/.ssh/cwvj_gha_ed25519

# Verify loaded SSH keys
ssh-add -l
```

**Sample output:**

```text
Agent pid 4821
Identity added: /home/you/.ssh/cwvj_gha_ed25519 (your-email@example.com)
256 SHA256:3vX1...q9 your-email@example.com (ED25519)
```

- `eval "$(ssh-agent -s)"` starts the agent and sets the environment variables your shell needs to talk to it. (`eval` runs the text that `ssh-agent -s` prints.)
- `ssh-add` loads the private key into the agent.
- `ssh-add -l` lists the loaded keys. If you see a **fingerprint**, it worked.

### Step 4: Add the public key to GitHub

```bash
cat ~/.ssh/cwvj_gha_ed25519.pub
```

Copy the output, then on GitHub go to:

```text
Profile Picture → Settings → SSH and GPG Keys → New SSH Key
```

- **Title:** `cwvj-gha-practice`
- **Key:** paste the public key

GitHub now trusts your laptop for SSH Git operations. Test it:

```bash
ssh -T git@github.com
```

```text
Hi your-user! You've successfully authenticated, but GitHub does not provide shell access.
```

Now copy the repo's SSH URL from **Repository → Code → SSH**. It looks like:

```text
git@github.com:<YOUR_GITHUB_USERNAME>/<YOUR_REPOSITORY_NAME>.git
```

### Step 5: Create the local repo, commit, and push

```bash
# create the project directory and move inside it
mkdir project-files
cd project-files

# initialize a local git repository (creates the hidden .git/ folder)
git init

# ensure the primary branch is named main
git branch -M main

# the folder where GitHub looks for workflow files
mkdir -p .github/workflows

# create a sample README file
touch README.md

# add all files to the staging area (the "about to be committed" list)
git add .

# create the first commit
git commit -m "ci: add initial GitHub Actions workflow"

# connect the local repo to GitHub (one-time setup)
git remote add origin git@github.com:<YOUR_GITHUB_USERNAME>/<YOUR_REPOSITORY_NAME>.git

# push main to GitHub and remember origin/main as its upstream
git push -u origin main
```

| Command | What it does |
|---|---|
| `git init` | Turns the folder into a Git repo by creating `.git/`. Now Git can track commits and branches. |
| `git branch -M main` | Renames the current branch to `main`. Some Git installations still default to `master`, but GitHub uses `main`. Matching names avoids confusion during pushes and PRs. `-M` means "rename, even if a branch of that name exists". |
| `mkdir -p .github/workflows` | GitHub **only** reads workflow files from this exact folder. `-p` creates parent folders too. |
| `git add .` | Stages every changed file. |
| `git commit -m "..."` | Saves a snapshot with a message. |
| `git remote add origin <url>` | Names the GitHub repo `origin`. One-time setup. |
| `git push -u origin main` | Sends commits. `-u` sets the **upstream** (the remote branch your local branch tracks), so later you can just type `git push`. Because no destination name is given, Git creates or updates a remote branch called `main`. To push to a different remote name, write both: `git push -u origin main:main-remote`. |

**Sample output of the push:**

```text
Enumerating objects: 4, done.
Writing objects: 100% (4/4), 312 bytes | 312.00 KiB/s, done.
To github.com:your-user/cwvj-gha-practice.git
 * [new branch]      main -> main
branch 'main' set up to track 'origin/main'.
```

A successful push proves that:

- SSH authentication works
- GitHub recognizes and trusts your machine
- the remote is configured correctly

### Step 6: The branch → PR flow you'll use in almost every lab

```bash
git checkout -b feature/my-change     # create and switch to a new branch
# ...edit files...
git add .
git commit -m "feat: my change"
git push -u origin feature/my-change  # push the branch and set its upstream
```

Then on GitHub: **Pull requests → New pull request → base: `main` ← compare: `feature/my-change` → Create**.

Each new commit you push to that branch **updates the PR**. Later you'll learn this is the PR's `synchronize` activity, and it re-runs your checks.

Cleaning up after a merge (used in the capstone):

```bash
git switch main                    # 1. switch to main
git pull                           # 2. sync local main with the remote
git fetch --prune                  # 3. remove stale remote-tracking references
git branch -d feature/my-change    # 4. delete the finished local branch
git branch                         # 5. check which local branches remain
```

---

## Hands-on lab

1. Create a private repo `cwvj-gha-practice` with no README.
2. Generate an Ed25519 key, load it into the agent, add the public key to GitHub, and run `ssh -T git@github.com`.
3. Run the Step 5 commands. Refresh the repo on GitHub and you should see `README.md`.
4. Create `feature/test`, change README, push it, and open a PR into `main`. Push one more commit and watch the PR update.
5. Keep this repo. You'll use it from topic 08 onwards.

---

## Where you'll use it

- **Every lab** in this course starts here.
- **Feature branch → PR → merge to `main`** is the standard company flow. Most CI runs on PRs, and most CD runs after a merge to `main`.
- **SSH keys and PATs** come up again for self-hosted runners and cross-repo access. In GitHub Actions itself you'll mostly use the built-in `GITHUB_TOKEN` instead (topic 18).

---

## When it breaks

| Error | What it means | Fix |
|---|---|---|
| `Permission denied (publickey).` | GitHub didn't accept any key. | Run `ssh-add -l`. If it says no identities, re-run `ssh-add`. Check that the `.pub` key is added on GitHub. |
| `remote: Repository not found.` | Wrong URL, or your account has no access to the private repo. | Re-copy the SSH URL and check the owner/name. |
| `error: src refspec main does not match any` | You have no commits yet, or your branch is still `master`. | Commit first, then run `git branch -M main`. |
| `! [rejected] main -> main (fetch first)` | The remote has commits you don't have (for example, you created the repo with a README). | `git pull --rebase origin main`, then push. |
| `fatal: remote origin already exists.` | You ran `git remote add` twice. | `git remote set-url origin <url>`. |
| Workflow never shows up in the Actions tab | The file is not in **exactly** `.github/workflows/` (typo, wrong case, or wrong folder). | Run `ls -R .github` to check. |

---

## Quick recap

- GitHub Actions is driven by Git events: push, branch, tag, PR.
- SSH = private key on your laptop + public key on GitHub.
- `git init` → `git branch -M main` → `git add` → `git commit` → `git remote add origin` → `git push -u origin main`.
- Workflows **must** live in `.github/workflows/`.
- Feature branch → PR → merge is the flow you'll automate.

**Interview questions**

1. **Q: Why doesn't GitHub Actions need a webhook like Jenkins does?**
   A: It's built into GitHub, so repository events reach it directly inside the platform.
2. **Q: What does `git push -u` do?**
   A: It pushes and sets the upstream tracking branch, so future `git push` and `git pull` know where to go.
3. **Q: Which part of an SSH key pair do you upload to GitHub?**
   A: Only the public key (`.pub`). The private key never leaves your machine.
