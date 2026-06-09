# merl-blog

Static Astro blog for `merl.one`.

## Write posts

- Add new posts under `src/content/post/` as `.md` or `.mdx` files.
- Add notes under `src/content/note/` if you want shorter, less formal entries.
- Use tags in frontmatter to group posts and generate tag pages automatically.

## Images

Use local assets for post images:

- Put an image next to the post markdown file, or in a nearby folder such as `src/content/post/my-post/cover.jpg`.
- Reference inline images with Markdown, for example `![Alt text](./cover.jpg)`.
- Use the `coverImage` frontmatter field when you want a hero image at the top of the post.
- Keep shared site-wide assets in `public/` only when multiple posts need the same file.

Recommended pattern:

```text
src/content/post/my-post.md
src/content/post/my-post/cover.jpg
src/content/post/my-post/screenshot.png
```

## Local workflow

```bash
npm install
npm run check
npm run build
```

## Deploy

The intended production workflow is Cloudflare Pages Git integration:

1. Push this repo to GitHub.
2. Connect the GitHub repo to a Cloudflare Pages project.
3. Set the build command to `npm run build`.
4. Set the output directory to `dist`.
5. Set the production branch to `main`.
6. Attach the custom domain `merl.one`.

Cloudflare will then build and deploy automatically on every push to `main`, and create preview deployments for branches and pull requests.

## GitHub Actions

The GitHub Actions workflow in `.github/workflows/ci.yml` runs `npm run check` and `npm run build` on pushes and pull requests.
It is there to catch regressions early. Cloudflare Pages should remain the deployment target.
