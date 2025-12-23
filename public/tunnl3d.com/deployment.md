# Tunnl3D Cloudflare Pages Deployment

## Prerequisites

- [Node.js](https://nodejs.org/) (v18 or later)
- [Wrangler CLI](https://developers.cloudflare.com/workers/wrangler/install-and-update/) installed globally
- Cloudflare account with Pages enabled

## Project Structure

```
public/tunnl3d.com/
├── wrangler.toml      # Cloudflare Pages configuration
├── _headers           # Custom HTTP headers
├── _redirects         # URL redirect rules
├── deployment.md      # This file
└── root/              # Static site content (build output)
    ├── index.html     # Holding page
    ├── styles.css     # Styles
    └── 404.html       # Custom 404 page
```

## Deployment Options

### Option 1: Wrangler CLI (Direct Deploy)

1. **Install Wrangler** (if not already installed):
   ```bash
   npm install -g wrangler
   ```

2. **Authenticate with Cloudflare**:
   ```bash
   wrangler login
   ```

3. **Deploy from project directory**:
   ```bash
   cd public/tunnl3d.com
   wrangler pages deploy root --project-name=tunnl3d-com
   ```

### Option 2: Cloudflare Dashboard (Git Integration)

1. Log in to [Cloudflare Dashboard](https://dash.cloudflare.com/)
2. Navigate to **Workers & Pages** → **Create application** → **Pages**
3. Connect your GitHub repository
4. Configure build settings:
   - **Project name**: `tunnl3d-com`
   - **Production branch**: `main`
   - **Build command**: *(leave empty - static site)*
   - **Build output directory**: `public/tunnl3d.com/root`
5. Click **Save and Deploy**

### Option 3: GitHub Actions (CI/CD)

Create `.github/workflows/deploy-pages.yml`:

```yaml
name: Deploy to Cloudflare Pages

on:
  push:
    branches: [main]
    paths:
      - 'public/tunnl3d.com/**'

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Deploy to Cloudflare Pages
        uses: cloudflare/wrangler-action@v3
        with:
          apiToken: ${{ secrets.CLOUDFLARE_API_TOKEN }}
          accountId: ${{ secrets.CLOUDFLARE_ACCOUNT_ID }}
          command: pages deploy public/tunnl3d.com/root --project-name=tunnl3d-com
```

**Required GitHub Secrets**:
- `CLOUDFLARE_API_TOKEN` - API token with Pages edit permissions
- `CLOUDFLARE_ACCOUNT_ID` - Your Cloudflare account ID

## Custom Domain Setup

1. In Cloudflare Dashboard, go to your Pages project
2. Navigate to **Custom domains** tab
3. Click **Set up a custom domain**
4. Enter `tunnl3d.com` and follow DNS configuration steps
5. Optionally add `www.tunnl3d.com` as an additional domain

## Configuration Files

### `_headers`
Defines custom HTTP response headers for security and caching.

### `_redirects`
Defines URL redirect rules. Format:
```
/old-path /new-path 301
```

### `wrangler.toml`
Wrangler CLI configuration for local development and deployment.

## Local Preview

```bash
cd public/tunnl3d.com
wrangler pages dev root
```

This starts a local server at `http://localhost:8788`.

## Useful Links

- [Cloudflare Pages Docs](https://developers.cloudflare.com/pages/)
- [Wrangler CLI Reference](https://developers.cloudflare.com/workers/wrangler/commands/)
- [Pages Configuration](https://developers.cloudflare.com/pages/configuration/)
