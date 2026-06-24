# Managing Multiple GitHub Deploy Keys on One Server 🔑

When you have multiple projects (e.g., a **Backend** and a **Frontend**) on a single VPS, GitHub won't allow you to use the same SSH key (Deploy Key) for both. You need a unique key for each and a configuration to handle them.

> **All commands below run on the VPS**, not on your local machine. The VPS holds the private keys; GitHub only ever sees the public keys. This follows the same VPS-first principle as [`server_deployment.md`](./server_deployment.md).

---

### 1️⃣ Step 1: Generate Unique Keys on the VPS

SSH into your VPS and generate a key per repository. Use the `-N ""` flag to set an empty passphrase — this is **crucial for automated deployments** (a passphrase would block non-interactive `git pull`).

**For Frontend:**

```bash
ssh-keygen -t ed25519 -C "frontend@example.com" -f ~/.ssh/id_ed25519_frontend -N ""
```

**For Backend:**

```bash
ssh-keygen -t ed25519 -C "backend@example.com" -f ~/.ssh/id_ed25519_backend -N ""
```

- `-t ed25519`: The most secure and modern key type.
- `-C`: A comment to identify the key.
- `-f`: The specific file path and name.
- `-N ""`: Sets an empty passphrase (crucial for automated deployments).

---

### 2️⃣ Step 2: Configure the SSH Config File

On your **VPS**, edit `~/.ssh/config` to define "Aliases" for each project. This tells SSH which key to use for which purpose.

```text
# Configuration for Backend Repository
Host github-backend
    HostName github.com
    User git
    IdentityFile ~/.ssh/id_ed25519_backend
    IdentitiesOnly yes
    BatchMode yes

# Configuration for Frontend Repository
Host github-frontend
    HostName github.com
    User git
    IdentityFile ~/.ssh/id_ed25519_frontend
    IdentitiesOnly yes
    BatchMode yes
```

#### What do these parameters mean?

- **Host**: This is your custom nickname (Alias). You will use `github-backend` instead of `github.com` in your git commands.
- **HostName**: The actual server to connect to (`github.com`).
- **User**: For GitHub SSH, this is always `git`.
- **IdentityFile**: The absolute path to the **Private Key** for this specific host.
- **IdentitiesOnly yes**: Forces SSH to ONLY use the key specified in `IdentityFile`, ignoring any other keys in the agent.
- **BatchMode yes**: Disables interactive prompts (like asking for a password). If the key fails, it stops immediately rather than waiting for user input.

---

### 3️⃣ Step 3: Cloning with the Aliases

When you clone your repositories, replace `github.com` with the **Host** alias you created in Step 2.

**For Backend:**

```bash
git clone git@github-backend:your-username/backend-repo.git
```

**For Frontend:**

```bash
git clone git@github-frontend:your-username/frontend-repo.git
```

---

### 4️⃣ Step 4: Security Checklist

1.  **Strict Permissions:** SSH will ignore keys if permissions are too open.
    ```bash
    chmod 700 ~/.ssh
    chmod 600 ~/.ssh/id_ed25519_backend
    chmod 600 ~/.ssh/id_ed25519_frontend
    ```
2.  **Add Public Keys to GitHub:** Don't forget to copy the content of `.pub` files to **GitHub Repo > Settings > Deploy keys**.
3.  **Test Connections:**
    ```bash
    ssh -T git@github-backend
    ssh -T git@github-frontend
    ```
    A successful response for each looks like:
    ```
    Hi <username>/<repo-name>! You've successfully authenticated, but GitHub does not provide shell access.
    ```

---

### 5️⃣ Step 5: Adapting Existing Repos Already Cloned with `github.com`

If a repo was previously cloned with the plain `github.com` URL, repoint it to the correct alias **on the VPS**:

```bash
# From inside the repo on the VPS
git remote set-url origin git@github-backend:your-username/backend-repo.git
```

Do this for each repo so the right key is used automatically.

---

### 📋 Quick Recap

| Item                       | Lives On         | Path / Location                          |
| :------------------------- | :--------------- | :--------------------------------------- |
| Private keys               | **VPS**          | `~/.ssh/id_ed25519_backend`, `~/.ssh/id_ed25519_frontend` |
| Public keys (locks)        | **GitHub**       | Each repo > Settings > Deploy keys       |
| SSH config aliases         | **VPS**          | `~/.ssh/config`                          |
| Cloned repos               | **VPS**          | `/var/www/<project>` (or your app path)  |
