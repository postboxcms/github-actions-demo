# github-actions-demo

Static site demo with GitHub Actions deployment to a DigitalOcean droplet over SSH.

## Workflow

[`.github/workflows/deploy-digitalocean.yml`](.github/workflows/deploy-digitalocean.yml) runs on every push to `main` and on manual **Run workflow** from the Actions tab. It checks out the repo and `rsync`s the project (excluding `.git` and `.github`) to your droplet.

## GitHub repository secrets

Create these under **Settings → Secrets and variables → Actions**:

| Secret | Description |
|--------|-------------|
| `DO_SSH_PRIVATE_KEY` | PEM private key that matches a public key in `~/.ssh/authorized_keys` on the droplet (full key including `-----BEGIN ...` lines). |
| `DO_HOST` | Droplet public IPv4/IPv6 or DNS name. |
| `DO_USER` | SSH login user (for example `root` or `deploy`). |
| `DO_DEPLOY_PATH` | *(Optional)* Absolute path on the server; defaults to `/var/www/html` if omitted. |

## Droplet preparation

1. Create an SSH key pair locally (or in a secure password manager flow); add the **public** key to the deploy user’s `~/.ssh/authorized_keys` on the droplet.
2. Ensure the deploy path exists and the SSH user can write there, for example:
   - `sudo mkdir -p /var/www/html && sudo chown "$USER:$USER" /var/www/html`
3. Point your web server (Nginx, Caddy, etc.) document root at that directory.

After secrets are set, push to `main` to deploy.
