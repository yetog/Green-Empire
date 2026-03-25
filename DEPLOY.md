# Green Empire — Deployment Brief for Portfolio Agent

**Target URL:** `https://green-empire.zaylegend.com`
**Repo:** `https://github.com/yetog/Green-Empire`
**Server user:** `langchain`
**Server nginx sites:** `/etc/nginx/sites-available/`
**Portfolio root:** `/var/www/zaylegend/portfolio/dist`

---

## Why a Subdomain (Not a Subpath)

The site uses root-absolute asset paths throughout every HTML file:
```html
<link rel="stylesheet" href="/css/main.css" />
<img src="/images/logo.png" />
<script src="/js/main.js"></script>
```

If served at `zaylegend.com/green-empire`, every asset request would go to
`zaylegend.com/css/main.css` — hitting the portfolio nginx, not the container.
The existing docker app manager rewrite trick only fixes the entry URL, not
these embedded absolute references in the HTML.

**Solution: serve it at a subdomain root** (`green-empire.zaylegend.com`),
where all absolute paths resolve correctly.

---

## Step 1 — Dockerfile (add to repo root)

Create a `Dockerfile` in the `green-empire` repo:

```dockerfile
FROM nginx:alpine
COPY . /usr/share/nginx/html
COPY nginx.conf /etc/nginx/conf.d/default.conf
EXPOSE 80
```

Create `nginx.conf` in the repo root:

```nginx
server {
    listen 80;
    root /usr/share/nginx/html;
    index index.html;

    # Clean URLs — try file, then directory index, then 404
    location / {
        try_files $uri $uri/ $uri/index.html =404;
    }

    # Cache static assets
    location ~* \.(css|js|png|jpg|jpeg|gif|ico|svg|woff|woff2)$ {
        expires 1y;
        add_header Cache-Control "public, immutable";
    }

    # Don't serve the source generator files
    location ~ \.(py|json|md|sh)$ {
        return 404;
    }
}
```

---

## Step 2 — Build & Run the Docker Container

SSH into the server as `langchain`, then:

```bash
# Clone the repo
cd /home/langchain
git clone https://github.com/yetog/Green-Empire.git green-empire
cd green-empire

# Build the Docker image
docker build -t green-empire:latest .

# Run the container (port 3010 — pick any free port)
docker run -d \
  --name green-empire \
  --restart unless-stopped \
  -p 3010:80 \
  green-empire:latest

# Verify it's running
docker ps | grep green-empire
curl -I http://localhost:3010
```

---

## Step 3 — Nginx Subdomain Config

Create a new nginx site config:

```bash
sudo nano /etc/nginx/sites-available/green-empire
```

Paste this:

```nginx
server {
    listen 80;
    server_name green-empire.zaylegend.com;

    location / {
        proxy_pass http://localhost:3010;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

Enable it and reload nginx:

```bash
sudo ln -s /etc/nginx/sites-available/green-empire /etc/nginx/sites-enabled/green-empire
sudo nginx -t
sudo systemctl reload nginx
```

---

## Step 4 — DNS Record

Add an A record in the DNS panel (IONOS or wherever the domain is managed):

| Type | Host | Value | TTL |
|------|------|-------|-----|
| A | `green-empire` | `<server IP>` | 300 |

Wait for propagation (usually 5–15 min with a short TTL).

---

## Step 5 — SSL Certificate

Once DNS is propagated:

```bash
sudo certbot --nginx -d green-empire.zaylegend.com
```

Certbot will automatically update the nginx config with the SSL cert. Choose
option 2 (redirect HTTP to HTTPS) when prompted.

---

## Step 6 — Verify

```bash
# Check the container is running
docker ps | grep green-empire

# Check nginx config is valid
sudo nginx -t

# Test the live URL
curl -I https://green-empire.zaylegend.com
curl -I https://green-empire.zaylegend.com/css/main.css
curl -I https://green-empire.zaylegend.com/services/lawn-mowing/
```

All three should return `200 OK`.

---

## Updating the Site Later

When the site is regenerated (e.g. Formspree ID is added, reviews updated):

```bash
cd /home/langchain/green-empire

# Pull latest from repo
git pull

# Rebuild and restart container
docker build -t green-empire:latest .
docker stop green-empire && docker rm green-empire
docker run -d \
  --name green-empire \
  --restart unless-stopped \
  -p 3010:80 \
  green-empire:latest
```

Or wrap this in a one-liner alias:

```bash
echo 'alias deploy-green-empire="cd /home/langchain/green-empire && git pull && docker build -t green-empire:latest . && docker stop green-empire && docker rm green-empire && docker run -d --name green-empire --restart unless-stopped -p 3010:80 green-empire:latest"' >> ~/.bashrc
source ~/.bashrc
```

Then `deploy-green-empire` redeploys in one command.

---

## What to Add to the Portfolio Showcase

Once live, add a project card on `zaylegend.com` pointing to:
- **Live demo:** `https://green-empire.zaylegend.com`
- **Repo:** `https://github.com/yetog/Green-Empire`
- **Description:** Client site for Green Empire Landscaping, Hempstead NY. Static site generator (Python) producing 41 SEO-optimized pages targeting Long Island landscaping searches. LocalBusiness schema, 20 city-specific landing pages, Formspree contact forms.

---

## Quick Reference

| Thing | Value |
|-------|-------|
| Live URL | `https://green-empire.zaylegend.com` |
| Container name | `green-empire` |
| Container port | `3010` |
| Docker image | `green-empire:latest` |
| Repo path on server | `/home/langchain/green-empire` |
| Nginx config | `/etc/nginx/sites-available/green-empire` |
| Client's next action | Replace `YOURCODE` in `site.config.json > brand.formspreeId`, push to repo, redeploy |
