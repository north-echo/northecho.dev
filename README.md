# northecho.dev

Personal site for security research and open-source tooling.

## Local Development

```bash
# Install Hugo (https://gohugo.io/installation/)
# macOS:
brew install hugo

# Run dev server
hugo server -D

# Build for production
hugo --minify
```

## Deployment

Pushes to `main` deploy to Cloudflare Pages via GitHub Actions.

### Setup

1. Create a Cloudflare Pages project named `northecho-dev`
2. Add repo secrets:
   - `CLOUDFLARE_API_TOKEN` — API token with Pages edit permissions
   - `CLOUDFLARE_ACCOUNT_ID` — your Cloudflare account ID
3. In Cloudflare DNS, add a CNAME record: `northecho.dev` → `northecho-dev.pages.dev`

## Writing Posts

```bash
hugo new posts/my-new-post.md
```

Edit the front matter, remove `draft: true` when ready, push to `main`.

## License

Content: [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/)
Code: [MIT](https://opensource.org/licenses/MIT)
