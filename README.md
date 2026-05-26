# github-actions-demo

Static site demo with GitHub Actions deployment to a DigitalOcean droplet over SSH.

## Workflow

[`.github/workflows/deploy-digitalocean.yml`](.github/workflows/deploy-digitalocean.yml) runs on every push to `main` and on manual **Run workflow** from the Actions tab. It checks out the repo and `rsync`s the project (excluding `.git` and `.github`) to your droplet.

## SSH key setup for `DO_SSH_PRIVATE_KEY`

Use a **dedicated deploy key** only for this repository and droplet. Do not reuse your personal day-to-day SSH key in GitHub Actions secrets.

### 1. Generate a new key pair (on your machine)

Open a terminal (PowerShell, macOS Terminal, or Linux shell). Use Ed25519:

```bash
ssh-keygen -t ed25519 -C "github-actions-deploy" -f ./do-github-actions-deploy -N ""
```

- `-f ./do-github-actions-deploy` creates `do-github-actions-deploy` (private) and `do-github-actions-deploy.pub` (public) in the current directory.
- `-N ""` sets an empty passphrase so GitHub Actions can use the key non-interactively. If you prefer a passphrase, omit `-N ""` and you would need a different automation approach; for typical deploy keys, empty passphrase on a **single-purpose** key is common.

If your environment does not support Ed25519, use RSA:

```bash
ssh-keygen -t rsa -b 4096 -C "github-actions-deploy" -f ./do-github-actions-deploy -N ""
```

### 2. Install the public key on the droplet

`ssh-copy-id` only works if **this computer can already SSH into the droplet** (it uses your normal SSH keys or password for the initial connection; the `-i ./do-github-actions-deploy.pub` argument is the key you are *installing*, not the key used to log in). On DigitalOcean, **password authentication is usually off**, so if you did not add your PC’s default `~/.ssh/id_*` key when you created the droplet, `ssh-copy-id` fails with `Permission denied (publickey)`.

**Prefer manual install** if you are unsure which key works, or use the web console (Option C).

**Option A — `ssh-copy-id` when you already have SSH access**

If logging in normally works (for example you use a specific key file):

```bash
ssh-copy-id -i ./do-github-actions-deploy.pub -o IdentitiesOnly=yes -o IdentityFile="$HOME/.ssh/YOUR_EXISTING_DO_KEY" YOUR_USER@YOUR_DROPLET_IP
```

Replace `YOUR_EXISTING_DO_KEY` with the private key file that actually opens the droplet (often the one you selected in the DigitalOcean create-droplet UI). On Windows PowerShell you can use a path like `$env:USERPROFILE\.ssh\YOUR_EXISTING_DO_KEY`.

**Option B — manual (reliable)**

1. Open a shell **from a machine that can SSH into the droplet** (same key as in the DigitalOcean UI, or use Option C).
2. Show the **new** deploy public key:

   ```bash
   cat ./do-github-actions-deploy.pub
   ```

3. SSH in as the user that will run deploys (`DO_USER`):

   ```bash
   ssh -o IdentitiesOnly=yes -i ~/.ssh/YOUR_EXISTING_DO_KEY YOUR_USER@YOUR_DROPLET_IP
   ```

4. On the droplet, append **one line** (the full line from the `.pub` file) to `authorized_keys`:

   ```bash
   mkdir -p ~/.ssh
   chmod 700 ~/.ssh
   nano ~/.ssh/authorized_keys
   # paste the new deploy public key as a new line, save, exit
   chmod 600 ~/.ssh/authorized_keys
   ```

   Or in one shot (paste between the single quotes):

   ```bash
   echo 'ssh-ed25519 AAAA...your-key... github-actions-deploy' >> ~/.ssh/authorized_keys
   chmod 600 ~/.ssh/authorized_keys
   ```

**Option C — DigitalOcean web console (no local SSH needed)**

1. In the [DigitalOcean control panel](https://cloud.digitalocean.com/), open your droplet → **Access** → **Launch Droplet Console** (or **Recovery console**).
2. Log in as `root` or the image default user (Ubuntu images often use `ubuntu` first; you may need `sudo -i` for root).
3. Run the same `mkdir` / `chmod` / append-to-`authorized_keys` commands as in Option B for the user that matches `DO_USER`.

**Wrong SSH user:** many images use `root` or `ubuntu`, not both. Using `root@` on an Ubuntu cloud image that only allows `ubuntu` can produce `Permission denied (publickey)`. Match the user DigitalOcean documents for your image.

#### `Permission denied (publickey)` on `ssh-copy-id`

| Cause | What to do |
|--------|------------|
| This PC does not have the droplet’s original key loaded | Use Option B with `-i ~/.ssh/the_key_you_added_in_DO`, or Option C. |
| Wrong username | Try `root` or `ubuntu` per your image; confirm in DigitalOcean droplet docs. |
| New deploy key used as login key | The deploy private key is not on the server yet—you must get in with an **existing** key or the console first. |

### 3. Verify SSH from your machine

```bash
ssh -i ./do-github-actions-deploy YOUR_USER@YOUR_DROPLET_IP
```

You should log in without a password prompt (only the key). Exit with `exit`.

### 4. Add the private key as `DO_SSH_PRIVATE_KEY` in GitHub

1. Open the repository on GitHub: **Settings → Secrets and variables → Actions**.
2. **New repository secret**.
3. Name: `DO_SSH_PRIVATE_KEY` (exact spelling).
4. Value: the **entire** private key file, including the first and last lines, for example:

   ```text
   -----BEGIN OPENSSH PRIVATE KEY-----
   ...multiple lines...
   -----END OPENSSH PRIVATE KEY-----
   ```

   Legacy RSA PEM uses `BEGIN RSA PRIVATE KEY` / `END RSA PRIVATE KEY`.

5. On Windows PowerShell you can copy the private key from disk:

   ```powershell
   Get-Content -Raw .\do-github-actions-deploy
   ```

6. Save the secret. **Do not** commit `do-github-actions-deploy` to git. Add the key filenames to `.gitignore` if you keep them in the repo folder.

### 5. Secure the key files locally

After the secret is saved and deploy works, delete the private key from your disk or move it to a password manager / secure vault if you need a backup. Anyone with the private key can SSH as that user to the droplet.

---

## Other GitHub repository secrets

Create these under **Settings → Secrets and variables → Actions**:


| Secret               | Description                                                                       |
| -------------------- | --------------------------------------------------------------------------------- |
| `DO_SSH_PRIVATE_KEY` | Full private key (see above).                                                     |
| `DO_HOST`            | Droplet public IPv4/IPv6 or DNS name.                                             |
| `DO_USER`            | SSH user that owns the authorized key (for example `root` or `deploy`).           |
| `DO_DEPLOY_PATH`     | *(Optional)* Absolute path on the server; defaults to `/var/www/html` if omitted. |


## Droplet preparation

1. Public key for `DO_USER` is in `~/.ssh/authorized_keys` (see step 2 above).
2. Deploy path exists and `DO_USER` can write there, for example:
  ```bash
   sudo mkdir -p /var/www/html && sudo chown "$USER:$USER" /var/www/html
  ```
3. Point your web server (Nginx, Caddy, etc.) document root at that directory.

After all secrets are set, push to `main` to deploy.