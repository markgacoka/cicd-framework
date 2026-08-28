# VPS Deployment Setup

## Architecture

```
GitHub Actions → SSH → VPS (89.116.157.98)

/var/www/app/
├── current/          ← symlink to active release (nginx serves this)
├── releases/
│   ├── a1b2c3d/      ← release by git SHA
│   ├── e4f5g6h/      ← previous release (kept for rollback)
│   └── ...           ← last 5 releases kept
└── .env              ← persisted env vars (never in git)
```

Deploys are atomic: the `current` symlink is swapped only after the new release is fully unpacked and dependencies installed. If the health check fails, the previous release is automatically restored.

## One-Time VPS Setup

SSH into the VPS and run:

```bash
ssh gacoka-vps

# Create deploy directory structure
sudo mkdir -p /var/www/app/releases
sudo chown -R gacoka:gacoka /var/www/app

# Create the .env file (populated manually — never committed to git)
touch /var/www/app/.env
chmod 600 /var/www/app/.env

# Install Node.js (if using Node)
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
sudo apt-get install -y nodejs

# Install PM2 (Node process manager)
sudo npm install -g pm2
pm2 startup  # follow the printed command to enable autostart

# OR for Python: install gunicorn
pip3 install gunicorn

# Configure nginx to serve from /var/www/app/current
sudo nano /etc/nginx/sites-available/gacoka.com
```

**nginx config:**
```nginx
server {
    listen 80;
    server_name gacoka.com www.gacoka.com;

    # Static files / SPA
    root /var/www/app/current/dist;  # or /out, /build, /public — adjust to your build output
    index index.html;

    # For APIs / Node server
    location /api {
        proxy_pass http://localhost:3000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_cache_bypass $http_upgrade;
    }

    # SPA fallback
    try_files $uri $uri/ /index.html;
}
```

```bash
sudo ln -s /etc/nginx/sites-available/gacoka.com /etc/nginx/sites-enabled/
sudo nginx -t && sudo systemctl reload nginx

# HTTPS (Let's Encrypt)
sudo apt install certbot python3-certbot-nginx -y
sudo certbot --nginx -d gacoka.com -d www.gacoka.com
```

## Adding GitHub Secrets

Go to: **github.com/markgacoka/cicd-framework → Settings → Secrets and variables → Actions**

| Secret | How to get it |
|---|---|
| `ANTHROPIC_API_KEY` | console.anthropic.com → API Keys (generate a new one) |
| `VPS_SSH_KEY` | See below — generate a dedicated deploy key |

### Generate the deploy SSH key

On your **Mac** (not the VPS):
```bash
# Generate a dedicated deploy key (no passphrase — Actions can't type one)
ssh-keygen -t ed25519 -f ~/.ssh/cicd_deploy -N "" -C "github-actions-deploy"

# Add the PUBLIC key to the VPS
ssh gacoka-vps "mkdir -p ~/.ssh && cat >> ~/.ssh/authorized_keys" < ~/.ssh/cicd_deploy.pub

# Copy the PRIVATE key — this is what goes into GitHub
cat ~/.ssh/cicd_deploy
```

Paste the private key output (starting with `-----BEGIN OPENSSH PRIVATE KEY-----`) into the `VPS_SSH_KEY` GitHub secret.

### PM2 ecosystem file (for Node apps)

Create `/var/www/app/current/ecosystem.config.js` in your project:
```js
module.exports = {
  apps: [{
    name: 'app',
    script: 'npm',
    args: 'start',
    cwd: '/var/www/app/current',
    env_production: {
      NODE_ENV: 'production',
      PORT: 3000,
    },
    max_restarts: 5,
    restart_delay: 3000,
  }]
}
```

## Deploy Workflow Reference

**Automatic:** Every merge to `main` triggers a deploy (after release workflow runs).

**Manual:**
```bash
gh workflow run deploy.yml --field environment=production
```

**Check deploy status:**
```bash
gh run list --workflow=deploy.yml --limit 5
gh run view <run-id> --log
```

**Rollback manually:**
```bash
ssh gacoka-vps
ls /var/www/app/releases/   # list available releases
ln -sfn /var/www/app/releases/<SHA> /var/www/app/current
pm2 restart all
```

## Adjusting the Deploy Path

The `DEPLOY_PATH` defaults to `/var/www/app`. To change it, set a **repository variable** (not secret):
> Settings → Secrets and variables → Actions → Variables → New variable
> Name: `DEPLOY_PATH` · Value: `/your/path`

Then update `deploy.yml`:
```yaml
env:
  DEPLOY_PATH: ${{ vars.DEPLOY_PATH || '/var/www/app' }}
```

## Health Check

The workflow calls `http://89.116.157.98` after deploy and retries 6× (every 10s, 60s total).
Update `HEALTHCHECK_URL` in `deploy.yml` once your domain is live:
```yaml
HEALTHCHECK_URL: https://gacoka.com/health
```

Add a `/health` endpoint to your app that returns `200 OK` when the app is ready.
