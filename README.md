# KnowYourToolset Blog

This repository is my personal blog that uses [Hugo](https://gohugo.io/) and I chose the [Clarity theme](https://themes.gohugo.io/themes/hugo-clarity/) from the [Hugo Themes Showcase](https://themes.gohugo.io/).

The way to run the site is:

``` bash
git submodule init
git submodule update
hugo serve
```

Then browse to [http://locahost:1313](http://localhost:1313)

## New Posts

To create a new post, use the command below and then make sure to update
the meta-data at the top of the file.  Remove the `draft: true` line
to have the post show up on the site.

``` bash
hugo new --kind post YYYY/MM/slug-for-your-new-article.md
```

Make sure to include the `YYYY` value in the `config\_default\params.toml` element called `mainSections`.
