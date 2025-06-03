# wyatthoho.github.io

## Introduction

This repository hosts my personal website.  
It is built using [Jekyll](https://jekyllrb.com/), 
following the official [step-by-step tutorial](https://jekyllrb.com/docs/step-by-step/01-setup/).

## Requirements

To build and serve the site locally, 
make sure the following are installed:

- [Ruby](https://www.ruby-lang.org/en/downloads/)
- RubyGems listed in the `Gemfile` (Jekyll is the main dependency)

Most modern Ruby installations come with **Bundler** preinstalled.  
Use Bundler to install the required gems:

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

