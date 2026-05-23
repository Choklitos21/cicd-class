# CI/CD SSH Setup for GitHub Actions and VPS Deployment

## Introduction

**SSH (Secure Shell)** is a cryptographic network protocol that allows secure communication between two computers over an unsecured network. Instead of passwords, SSH uses a **key pair** — a public key and a private key — to authenticate connections without exposing credentials.

In a CI/CD pipeline, SSH keys are essential because:

- **Passwords are insecure for automation** — they can't be typed interactively in a pipeline.
- **SSH keys are cryptographically strong** — virtually impossible to brute-force.
- **They integrate cleanly with secret managers** like GitHub Secrets, keeping credentials out of source code.

When a GitHub Actions workflow runs, it uses the **private SSH key** stored in GitHub Secrets to authenticate against your VPS, which holds the corresponding **public key** in its `authorized_keys` file. If the keys match, access is granted — fully automated, fully secure.

---

## Architecture Overview

```
┌─────────────────────────────────────────────┐
│              Local Machine                  │
│         Generate SSH Key Pair               │
└────────────────┬────────────────────────────┘
                 │
        ┌────────┴────────┐
        ▼                 ▼
┌──────────────┐   ┌──────────────────────┐
│  Public Key  │   │    Private Key       │
│  → VPS       │   │  → GitHub Secrets    │
│  authorized_ │   │  (SSH_PRIVATE_KEY)   │
│  keys        │   │                      │
└──────┬───────┘   └──────────┬───────────┘
       │                      │
       │          ┌───────────▼───────────┐
       │          │    GitHub Actions     │
       │          │    Workflow Trigger   │
       │          └───────────┬───────────┘
       │                      │
       │              SSH Connection
       │◄─────────────────────┘
       │        (Key Pair Match)
       ▼
┌─────────────────────────────────────────────┐
│                VPS Server                   │
│           Docker Deployment                 │
│         Updated Application Live            │
└─────────────────────────────────────────────┘
```

---

## Section 1 — Generate SSH Keys

### What Are Public and Private Keys?

SSH uses **asymmetric cryptography**: two mathematically linked keys are generated together.

| Key | Description | Where It Goes |
|-----|-------------|---------------|
| **Private Key** | Secret key that stays on your machine (or in GitHub Secrets). Never shared. | Your local machine / GitHub Secrets |
| **Public Key** | Safe to share. Placed on any server you want to access. | VPS `authorized_keys` file |

When GitHub Actions tries to connect, the server uses the public key to verify the private key signature — without ever transmitting the private key itself.

### Why ED25519 is Recommended

**ED25519** is a modern elliptic-curve algorithm that offers:

- ✅ Stronger security than RSA 2048-bit
- ✅ Shorter key length (faster operations)
- ✅ Better resistance to side-channel attacks
- ✅ Widely supported on modern systems (OpenSSH 6.5+)

### Generate an ED25519 Key (Recommended)

```bash
ssh-keygen -t ed25519 -C "github-actions-deploy"
```

### Alternative: RSA 4096-bit (for legacy compatibility)

```bash
ssh-keygen -t rsa -b 4096 -C "github-actions-deploy"
```

### Parameter Reference

| Parameter | Value | Description |
|-----------|-------|-------------|
| `-t` | `ed25519` / `rsa` | Specifies the key **type** (algorithm) |
| `-b` | `4096` | Key **bit size** (RSA only; larger = stronger) |
| `-C` | `"github-actions-deploy"` | A **comment** label to identify the key's purpose |

### Terminal Example

```
$ ssh-keygen -t ed25519 -C "github-actions-deploy"

Generating public/private ed25519 key pair.
Enter file in which to save the key (/home/youruser/.ssh/id_ed25519):
Enter passphrase (empty for no passphrase):
Enter same passphrase again:
Your identification has been saved in /home/youruser/.ssh/id_ed25519
Your public key has been saved in /home/youruser/.ssh/id_ed25519.pub
The key fingerprint is:
SHA256:xXzAbCdEfGhIjKlMnOpQrStUvWxYz1234567890 github-actions-deploy
```

> **Tip:** For CI/CD automation, leave the passphrase **empty** when prompted. A passphrase would require manual input during the pipeline run, breaking automation.

---

## Section 2 — SSH Key Storage Locations

After generation, your keys are stored in the `.ssh` directory of your home folder.

### Linux / macOS

```
~/.ssh/
```

### Windows (PowerShell)

```
C:\Users\YOUR_USER\.ssh\
```

### Key Files Explained

| File | Description |
|------|-------------|
| `id_ed25519` | **Private key** — keep this secret, never share |
| `id_ed25519.pub` | **Public key** — safe to copy to servers |
| `known_hosts` | List of servers your machine has connected to before |
| `authorized_keys` | (On servers) List of public keys allowed to connect |

### Security Considerations

- The `.ssh/` directory should be readable **only by your user** (`chmod 700`).
- The private key file should be readable **only by your user** (`chmod 600`).
- On shared or multi-user systems, incorrect permissions will cause SSH to refuse the key entirely.

> ### ⚠️ WARNING
>
> **Never share your private key (`id_ed25519`).**
> Anyone who has your private key can authenticate as you to any server that holds the corresponding public key. Treat it like a password — but more sensitive, because it cannot be changed with a simple reset.

---

## Section 3 — View and Copy the Public Key

To display your public key in the terminal, run:

```bash
cat ~/.ssh/id_ed25519.pub
```

### Expected Output

```
ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIGexamplekeycontenthereFooBarBazQuxQuux github-actions-deploy
```

Copy the **entire output** — from `ssh-ed25519` all the way to the comment at the end — as a single unbroken line. This is your public key and it must be pasted exactly as-is into the VPS.

The public key is used by your VPS to verify incoming connection requests. When GitHub Actions connects using the private key, the server checks it against this stored public key. If they match cryptographically, access is granted.

---

## Section 4 — Configure the VPS

### Step 1: Connect to Your VPS

```bash
ssh root@YOUR_SERVER_IP
```

### Step 2: Create the `.ssh` Directory

If it doesn't already exist, create it:

```bash
mkdir -p ~/.ssh
```

### Step 3: Add the Public Key to `authorized_keys`

Open (or create) the `authorized_keys` file:

```bash
nano ~/.ssh/authorized_keys
```

Paste your **full public key** on a new line. Save and exit (`Ctrl+O`, `Enter`, `Ctrl+X` in nano).

### How SSH Authorization Works

When a client connects, the server checks `~/.ssh/authorized_keys` for a public key that corresponds to the private key the client is presenting. If a match is found, authentication succeeds — no password required.

### Step 4: Set Correct Permissions

```bash
chmod 700 ~/.ssh
chmod 600 ~/.ssh/authorized_keys
```

### Why Permissions Matter

SSH is intentionally strict about file permissions. If your `.ssh` directory or `authorized_keys` file is readable by other users on the system, SSH will **refuse to use the keys** as a security measure. This prevents other users on a shared server from injecting unauthorized keys.

| Path | Required Permission | Meaning |
|------|---------------------|---------|
| `~/.ssh/` | `700` | Owner: read/write/execute. Group & others: no access |
| `~/.ssh/authorized_keys` | `600` | Owner: read/write. Group & others: no access |

---

## Section 5 — Test the SSH Connection

From your **local machine**, verify the key-based authentication works before configuring the pipeline:

```bash
ssh root@YOUR_SERVER_IP
```

### Expected Successful Behavior

```
Welcome to Ubuntu 22.04.3 LTS (GNU/Linux 5.15.0-91-generic x86_64)

Last login: Fri May 22 10:34:12 2026 from 192.168.1.100
root@your-vps:~#
```

If you connect **without being asked for a password**, the SSH key authentication is working correctly. You're now ready to configure GitHub Actions.

> **Troubleshooting:** If you're denied access, double-check:
> - The public key was pasted correctly (one line, no line breaks)
> - File permissions are `700` for `.ssh/` and `600` for `authorized_keys`
> - The SSH daemon on the VPS allows public key authentication (`PubkeyAuthentication yes` in `/etc/ssh/sshd_config`)

---

## Section 6 — Configure GitHub Secrets

### Why Secrets Are Necessary

GitHub Actions workflows run in ephemeral virtual environments. You cannot hardcode credentials in your workflow YAML because:

- The repository may be public (anyone can read it).
- Even in private repos, credentials in source code are a serious security risk.
- Leaked credentials in Git history are difficult to fully remove.

**GitHub Secrets** stores encrypted key-value pairs that are injected as environment variables at runtime — never visible in logs or source code.

### Navigation Path

```
Repository → Settings → Secrets and variables → Actions → New repository secret
```

### Required Secrets

| Secret Name | Value | Description |
|-------------|-------|-------------|
| `SERVER_IP` | `203.0.113.42` | The public IP address of your VPS |
| `SERVER_USER` | `root` or `deploy` | The SSH user to log in as on the VPS |
| `SSH_PRIVATE_KEY` | *(full contents of `id_ed25519`)* | The private key used for SSH authentication |

All three secrets are referenced in the workflow YAML using the `${{ secrets.SECRET_NAME }}` syntax.

---

## Section 7 — Copy the Private Key

To display the full private key for copying into GitHub Secrets:

```bash
cat ~/.ssh/id_ed25519
```

### Expected Output

```
-----BEGIN OPENSSH PRIVATE KEY-----
b3BlbnNzaC1rZXktdjEAAAAABG5vbmUAAAAEbm9uZQAAAAAAAAABAAAAMwAAAAtzc2gtZW
QyNTUxOQAAACBexamplekeycontentAAAAIHRoaXNpc2FuZXhhbXBsZXByaXZhdGVrZXli
... (many more lines) ...
-----END OPENSSH PRIVATE KEY-----
```

Copy the **entire output** — including the `-----BEGIN OPENSSH PRIVATE KEY-----` and `-----END OPENSSH PRIVATE KEY-----` header/footer lines. Paste this complete block as the value for the `SSH_PRIVATE_KEY` GitHub Secret.

### Why It Must Remain Secret

The private key is the sole proof of identity used to authenticate. Whoever possesses the private key can:

- Log in to any server that has the corresponding public key
- Execute arbitrary commands with the permissions of that user
- Potentially access other connected systems and data

> ### 🔐 Security Warning
>
> - **Never paste your private key into a chat, email, or issue tracker.**
> - **Never commit your private key to a Git repository** — even "accidentally" and then deleted. Git history is permanent and public.
> - **Rotate your keys immediately** if you suspect the private key has been exposed. Generate a new pair, update the VPS `authorized_keys`, and replace the GitHub Secret.
> - Store a backup of your private key only in an **encrypted password manager** (e.g., 1Password, Bitwarden).

---

## Section 8 — GitHub Actions Workflow Example

Create the file `.github/workflows/deploy.yml` in your repository:

```yaml
name: Deploy to VPS

on:
  push:
    branches:
      - main

jobs:
  deploy:
    name: SSH Deploy via Docker
    runs-on: ubuntu-latest

    steps:
      - name: Checkout repository
        uses: actions/checkout@v4

      - name: Deploy to VPS via SSH
        uses: appleboy/ssh-action@v1.0.3
        with:
          host: ${{ secrets.SERVER_IP }}
          username: ${{ secrets.SERVER_USER }}
          key: ${{ secrets.SSH_PRIVATE_KEY }}
          script: |
            # Pull the latest Docker image from the registry
            docker pull ghcr.io/${{ github.repository }}:latest

            # Stop and remove the existing container (if running)
            docker stop app-container || true
            docker rm app-container || true

            # Start the updated container
            docker run -d \
              --name app-container \
              --restart unless-stopped \
              -p 80:3000 \
              ghcr.io/${{ github.repository }}:latest

            echo "✅ Deployment complete."
```

### Workflow Breakdown

| Step | Action | Description |
|------|--------|-------------|
| `on: push: branches: [main]` | Trigger | Runs automatically on every push to `main` |
| `actions/checkout@v4` | Checkout | Clones the repo into the runner environment |
| `appleboy/ssh-action@v1.0.3` | SSH Action | Connects to VPS using the stored secrets |
| `docker pull` | Pull image | Downloads the latest image from the container registry |
| `docker stop / rm` | Cleanup | Gracefully stops and removes the old container |
| `docker run -d` | Deploy | Starts the new container in detached (background) mode |

> **Note:** Replace `ghcr.io/${{ github.repository }}:latest` with your actual Docker image path (Docker Hub, GHCR, ECR, etc.).

---

## Section 9 — Complete Deployment Flow

```
┌─────────────────────────────────────┐
│           Developer                 │
│      git push origin main           │
└────────────────┬────────────────────┘
                 │
                 ▼
┌─────────────────────────────────────┐
│          GitHub Actions             │
│   Workflow triggered on push        │
│   Loads SSH_PRIVATE_KEY secret      │
└────────────────┬────────────────────┘
                 │
                 ▼
┌─────────────────────────────────────┐
│        SSH Authentication           │
│  Private key ↔ Public key match     │
│  Secure tunnel established          │
└────────────────┬────────────────────┘
                 │
                 ▼
┌─────────────────────────────────────┐
│            VPS Server               │
│   Remote commands execute           │
└────────────────┬────────────────────┘
                 │
                 ▼
┌─────────────────────────────────────┐
│        Docker Deployment            │
│   Pull → Stop → Remove → Run        │
└────────────────┬────────────────────┘
                 │
                 ▼
┌─────────────────────────────────────┐
│       Updated Application           │
│     Live and serving traffic  🚀    │
└─────────────────────────────────────┘
```

---

## Section 10 — Best Practices

Following these practices will keep your CI/CD pipeline secure and maintainable:

| # | Practice | Why It Matters |
|---|----------|----------------|
| 1 | **Use ED25519 keys** | More secure and efficient than RSA; modern standard |
| 2 | **Avoid the `root` user in production** | A compromised root session grants full server control; use a dedicated `deploy` user with limited permissions |
| 3 | **Rotate SSH keys regularly** | Limits exposure window if a key is silently compromised; recommended every 6–12 months |
| 4 | **Restrict firewall access (port 22)** | Use a firewall (e.g., `ufw`, security groups) to allow SSH only from GitHub Actions IP ranges or a bastion host |
| 5 | **Never commit secrets to Git** | Even in private repos, hardcoded secrets are a critical vulnerability; always use GitHub Secrets or a vault |
| 6 | **Use separate deploy keys per project** | Don't reuse the same SSH key across multiple repositories or environments |
| 7 | **Monitor SSH login activity** | Review `/var/log/auth.log` (Linux) periodically for unexpected authentication attempts |
| 8 | **Keep the `appleboy/ssh-action` version pinned** | Avoid using `@master`; pin to a specific version tag to prevent supply-chain attacks |

---

## Final Notes

With this setup complete, your deployment pipeline is fully automated. Every time a developer pushes code to the `main` branch, GitHub Actions:

1. Detects the push event
2. Loads the SSH private key from GitHub Secrets
3. Opens a secure connection to the VPS
4. Executes the Docker deployment commands remotely
5. Serves the updated application — without any manual intervention

The entire deployment lifecycle — from code change to live production — is triggered by a single command:

```bash
git push
```

This is the power of CI/CD: reliable, repeatable, and fully automated deployments that free developers to focus on building features rather than managing servers.

---

*For questions or improvements, open an issue or submit a pull request.*