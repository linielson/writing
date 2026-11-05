# Writing

Articles by Linielson Rosa, published on [dev.to](https://dev.to). This repository is the source of truth: dev.to is
where the posts are read, and this is where they are written, reviewed and versioned.

## How it works

- Every post is a Markdown file in `posts/`, with dev.to front matter at the top (`title`, `published`, `description`,
  `tags`).
- A push to `main` that changes `posts/` runs `.github/workflows/publish.yml`, which uses
  [publish-devto](https://github.com/sinedied/publish-devto) to create or update the article on dev.to.
- A new post is created on dev.to as a **draft** while its front matter says `published: false`. Check the preview on
  dev.to, then set `published: true` and push again to publish it.
- After the first push, the workflow writes the dev.to `id` back into the post's front matter. From then on, pushes
  update that article instead of creating a new one. Don't remove the `id`.

## Writing a new post

1. Create `posts/<slug>.md` starting with:

   ```yaml
   ---
   title: "The title"
   published: false
   description: "One or two sentences for the preview card."
   tags: ruby, rails
   ---
   ```

   dev.to accepts up to four tags, lowercase, letters and numbers only.

2. Push to `main`, check the draft on dev.to, then flip `published` to `true` and push.

Editing the article on dev.to directly works, but the next push from here overwrites it. Edit here.

## Setup

The workflow needs a dev.to API key in the repository secret `DEVTO_TOKEN` (dev.to → Settings → Extensions → DEV
Community API Keys). Without it, the publish step is skipped.

## Posts

| Post                                                                                                         | Topic                                |
| ------------------------------------------------------------------------------------------------------------ | ------------------------------------ |
| [44 Rails advisories broke my CI on a PR that didn't touch a single gem](posts/rails-security-advisories.md) | Ruby, Rails, dependency security, CI |

Code for the projects mentioned in the posts: [Ledgerly](https://github.com/linielson/ledgerly).
