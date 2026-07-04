# Awesome Edwin

Static site for `awesomeedwin.com`, deployed to Cloudflare Pages with Wrangler.

## Deploy

```bash
npm install
npm run deploy
```

Wrangler uses `wrangler.jsonc`, which deploys the `public/` directory to the `awesomeedwin` Pages project.

## Custom domain & full deployment docs

`awesomeedwin.com` is live as a custom domain on the Pages project. For the
complete setup (auth gotchas, custom-domain API steps, troubleshooting), see
[DEPLOYMENT.md](DEPLOYMENT.md).
