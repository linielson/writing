---
title: 44 Rails advisories broke my CI on a PR that didn't touch a single gem
published: false
description: 'New advisories for Rails 8.1, rack and nokogiri turned my build red with no code change. What I found, how I fixed it, and how to make your CI catch it on its own.'
tags: 'ruby, rails, security, devops'
id: 4983552
---

I was finishing the authentication PR for [Ledgerly](https://github.com/linielson/ledgerly), an invoicing app I'm building in the open with Rails 8. Tests green, lint green. Then the security scan failed. The PR didn't add or update any gem.

It failed again two days later, on a different commit, for a different reason.

Nothing was broken in my code. What changed was the list of known vulnerabilities: new advisories had come out for gems the app was already using. The scan compares your `Gemfile.lock` against a public database that gets updated every day, so a build can go red without anyone touching anything.

## What it found

I ran `bundler-audit` against the lockfile from before the fix: 44 advisories across 11 gems. The worst ones were in Rails itself, all fixed in Rails 8.1.4.1:

- **Critical**, Active Storage: possible arbitrary file write and read during image analysis ([GHSA-f3w6-w445-fwh9](https://github.com/rails/rails/security/advisories/GHSA-f3w6-w445-fwh9))
- **High**, Action Pack: possible Strong Parameters bypass in `Parameters#each_key` ([GHSA-wm3x-v3qw-4jh3](https://github.com/rails/rails/security/advisories/GHSA-wm3x-v3qw-4jh3))
- **High**, Action Dispatch: possible IP spoofing in `RemoteIp` ([GHSA-2gff-2mmg-m73p](https://github.com/rails/rails/security/advisories/GHSA-2gff-2mmg-m73p))
- **High**, Active Support: possible denial of service in `safe_constantize` ([GHSA-8474-xrp6-wh69](https://github.com/rails/rails/security/advisories/GHSA-8474-xrp6-wh69))

The `RemoteIp` one hit close to home. The PR I was working on added a sign-in rate limit: 10 attempts every 3 minutes, counted per IP. If someone can fake the IP Rails sees, every request gets a fresh counter and the limit is useless.

The rest were in gems that come in through Rails and Kamal: rack, nokogiri, loofah, rails-html-sanitizer, crass and net-ssh. Several XSS and sanitizer bypasses, and a few ways to exhaust memory.

## The fix

It took about five minutes.

First, check with a fresh copy of the advisory database:

```sh
bin/bundler-audit check --update
```

On Rails 8.1 that binstub comes with `rails new` and reads `config/bundler-audit.yml`. On older apps, use `bundle exec bundler-audit check --update`.

The `--update` part caught me. My local run said "No vulnerabilities found" while CI was failing, because my copy of the database was weeks old. CI downloads a new one on every run.

Then update only what was flagged:

```sh
bundle update --conservative rails
bundle update --conservative rack nokogiri loofah rails-html-sanitizer crass net-ssh
```

Without `--conservative`, Bundler is free to move every gem it can. With it, only the gems you name move, plus whatever they strictly need.

Before committing, I read the `Gemfile.lock` diff. Every Rails gem went from 8.1.4 to 8.1.4.1, the others moved by a patch version, and nothing else changed. Tests passed, and it went out with the next deploy.

## Making CI catch it on its own

The part worth keeping is not the fix. It's that this can happen any day, with no code change, and you want CI to tell you before an attacker does.

If you're on Rails 8.1 or newer, you probably have this already: `rails new` generates a GitHub Actions workflow with a `scan_ruby` job that runs `bin/bundler-audit`. Two things to check: that the job is still there, and that a failure actually blocks the merge (in the repository settings, under Rules, require the `scan_ruby` check).

On an older app, add the gem:

```ruby
# Gemfile
group :development, :test do
  gem "bundler-audit", require: false
end
```

And a step in your CI job:

```yaml
- uses: ruby/setup-ruby@v1
  with:
    bundler-cache: true
- run: bundle exec bundler-audit check --update
```

Then the gap I had: my CI only ran on pull requests and pushes to `main`. In a quiet week with no PRs, a new advisory would sit there unnoticed. So I added a weekly run, and a button to run it by hand:

```yaml
on:
  pull_request:
  push:
    branches: [main]
  schedule:
    - cron: "0 9 * * 1" # Mondays, 09:00 UTC
  workflow_dispatch:
```

My workflow also deploys, so I double-checked that a scheduled run can't ship anything. The deploy job only runs on a push to `main`:

```yaml
deploy:
  if: github.event_name == 'push' && github.ref == 'refs/heads/main'
```

Two things about scheduled workflows I didn't know before reading the [GitHub docs](https://docs.github.com/en/actions/reference/workflows-and-actions/events-that-trigger-workflows):

- The failure email goes to whoever last changed the `cron` line, not necessarily the repository owner.
- In a public repository, GitHub turns scheduled workflows off after 60 days without activity. You turn them back on from the Actions tab.

Last, let GitHub do some of the work. In the repository settings, under Code security, turn on Dependabot alerts and Dependabot security updates. When an advisory affects your lockfile, it opens a PR with the patched version, and your CI tests it like any other change.

## What I'd tell myself a week ago

A red security scan on a PR that didn't touch dependencies isn't a flaky build. Refresh the database before trusting a local "all clear", update only what was flagged, read the lockfile diff, and don't let the scan depend on someone opening a PR.

The workflow and the notes behind this post are in the repo: [`.github/workflows/ci.yml`](https://github.com/linielson/ledgerly/blob/main/.github/workflows/ci.yml) and [`docs/ci.md`](https://github.com/linielson/ledgerly/blob/main/docs/ci.md).

**Sources**

- Rails security advisories: https://github.com/rails/rails/security/advisories
- bundler-audit: https://github.com/rubysec/bundler-audit
- Ruby advisory database: https://github.com/rubysec/ruby-advisory-db
- GitHub Actions `schedule` event: https://docs.github.com/en/actions/reference/workflows-and-actions/events-that-trigger-workflows
