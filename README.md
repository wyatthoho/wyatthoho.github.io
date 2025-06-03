# wyatthoho.github.io

## Introduction

This repository hosts my personal website: https://wyatthoho.github.io/.

It is built using [Jekyll][jekyll],
following the official [step-by-step tutorial][tutorial].

## Requirements

To build and serve the site locally,
make sure the following are installed:

- [Ruby][ruby_download]
- RubyGems (Jekyll is the main dependency)

Most modern Ruby distributions include **Bundler** by default.
For more details, see the [Bundler getting started guide][bundler].
After installing Ruby, use Bundler to install the necessary gems 
listed in the `Gemfile` :

```bash
bundle install
```

## Running Locally

To build and preview the site on your local machine:

```bash
bundle exec jekyll build
bundle exec jekyll serve
```

Then visit: http://127.0.0.1:4000

## Deployment

After committing and pushing to GitHub,
the default GitHub Actions workflow will be triggered automatically.
This will build the site and deploy it to GitHub Pages.

[jekyll]: https://jekyllrb.com/
[tutorial]: https://jekyllrb.com/docs/step-by-step/01-setup/
[ruby_download]: https://www.ruby-lang.org/en/downloads/
[bundler]: https://bundler.io/guides/getting_started.html#getting-started