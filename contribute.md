---
layout: page
title: Contribute
site_nav_category: contribute
site_nav_category_order: 400
is_site_nav_category: true
---

Contributions to this website are welcome and appreciated!

To contribute to this website, feel free to create pull requests for small fixes. For bigger contributions we recommend to start an issue on the [issue tracker](https://github.com/android/kotlin-guides/issues) first.

Pull requests should be made targeting the `master` branch. Every few weeks, the [change log](changelog.html) will be updated and all changes in that time period will be released to the `gh-pages` branch.

**We are looking forward to all of your contributions!**


## Development

You can run the site locally on your computer while making changes.

Ensure that you have Ruby and [Bundler](http://bundler.io/) installed.

### One-time setup

    bundle install --path vendor/bundle

_Note: If you're on Mac OS and this fails installing nokogiri, run `brew unlink xz`, install, and then `brew link xz`._

### Running the site

    bundle exec jekyll serve

Point your browser at [http://127.0.0.1:4000/kotlin-guides/](http://127.0.0.1:4000/kotlin-guides/).
