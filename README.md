# One doc repo HOW-TO guide

This repo can be used as a QuickStart template used to generate static site of just **ONE documentation**.

For more complex use cases, start with [Jekyll](https://jekyllrb.com/docs/).

This repo uses [Jekyll](https://jekyllrb.com/docs/) to generate static site. Jekyll is highly integrated with GitHub, see details at [GitHub Pages & Jekyll](https://docs.github.com/en/pages/setting-up-a-github-pages-site-with-jekyll/about-github-pages-and-jekyll).

This repo have:

* Jekyll setup
* [Primer theme for GitHub pages](https://github.com/pages-themes/primer) configured
* HTML elements supported: `<details>` and `<summary>`
* Local debug ready

## How to develop the documentation locally

Make sure you have the latest RubyGem, then do:

```bash
gem update bundle
bundle install
bundle exec jekyll build
bundle exec jekyll server
```

Go to `http://127.0.0.1:4000/` to see your doc.
