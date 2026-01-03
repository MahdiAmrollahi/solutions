# Comprehensive Deployment Guide 🚀

This document provides a precise roadmap for where to generate and where to store each SSH key.

### 🔑 Quick Summary Table

| Key Name              | Generated On | Private Key Location (Secret)          | Public Key Location (Lock)               |
| :-------------------- | :----------- | :------------------------------------- | :--------------------------------------- |
| **vps_deploy_key**    | **Local PC** | **GitHub Secrets** (`SSH_PRIVATE_KEY`) | **VPS** (`~/.ssh/authorized_keys`)       |
| **github_deploy_key** | **Local PC** | **VPS** (`~/.ssh/id_ed25519`)          | **GitHub Repo** (Settings > Deploy Keys) |

---

### 1️⃣ Connection A: GitHub Actions ➔ VPS

**Purpose:** Allows GitHub to "log into" your server and run commands (like `git pull`).

1.  **Generate on your LOCAL MACHINE:**
    ```bash
    ssh-keygen -t ed25519 -f ~/.ssh/vps_deploy_key -C "github-to-vps"
    ```
2.  **Where to put the Private Key:**
    - Open `~/.ssh/vps_deploy_key` on your local PC.
    - Copy the entire text.
    - Go to your **GitHub Repository > Settings > Secrets and variables > Actions**.
    - Create a new secret named `SSH_PRIVATE_KEY` and paste the text there.
3.  **Where to put the Public Key:**
    - Open `~/.ssh/vps_deploy_key.pub` on your local PC.
    - Log into your **VPS**.
    - Append the content to the file: `~/.ssh/authorized_keys`.
    - _Tip:_ You can use `ssh-copy-id -i ~/.ssh/vps_deploy_key user@your-vps-ip` to do this automatically.

---

### 2️⃣ Connection B: VPS ➔ GitHub Repo

**Purpose:** Allows the VPS to "download" (pull) the code from your private GitHub repository.

1.  **Generate on your LOCAL MACHINE (Recommended for backup) OR on the VPS:**
    _Standard practice is generating on local and moving it, but you can also generate it directly on the VPS._
    ```bash
    ssh-keygen -t ed25519 -f ~/.ssh/github_deploy_key -C "vps-to-github"
    ```
2.  **Where to put the Private Key:**
    - This must live on the **VPS** so it can identify itself to GitHub.
    - Save it as: `~/.ssh/id_ed25519` on the **VPS**.
    - **CRITICAL:** Set strict permissions on the VPS:
      ```bash
      chmod 600 ~/.ssh/id_ed25519
      ```
3.  **Where to put the Public Key:**
    - Open `~/.ssh/github_deploy_key.pub`.
    - Go to your **GitHub Repository > Settings > Deploy keys**.
    - Click **Add deploy key**, give it a title (e.g., "Production VPS"), and paste the public key.
    - _Security Note:_ Keep "Allow write access" **unchecked**.

---

### 3️⃣ Finishing Touches on VPS

To make sure everything works smoothly, run these commands on your **VPS**:

```bash
# Ensure the .ssh directory exists and has correct permissions
mkdir -p ~/.ssh
chmod 700 ~/.ssh

# If you haven't already, add the public key to authorized_keys
# (Only if you manually copied it)
chmod 600 ~/.ssh/authorized_keys

# Test if the VPS can talk to GitHub
ssh -T git@github.com
```

### 💡 Pro Tip: The SSH Config File

On your **VPS**, create a file at `~/.ssh/config` to force SSH to use the correct key for GitHub:

```text
Host github.com
  HostName github.com
  User git
  IdentityFile ~/.ssh/id_ed25519
  IdentitiesOnly yes
```
