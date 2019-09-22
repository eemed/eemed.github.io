---
title: Choosing and setting up a blog
date: 21.09.2019 16:41
tags: ['hexo', 'setup']
---

## Requirements

I wanted to write the posts with vim and effortlessly publish my posts. Latex support was also a must. The fist challenge was to choose the right platform. I wasn't familiar with all the blogging platforms / static site generators out there. I began researching. All of the options seemed to be good and have a large variety of themes.

### Options

1. [Hexo](https://hexo.io/) was my first find. It seemed simple and powerful. The only negative was that a large portion of the community is Chinese so the English documentation may not be up to date.
2. [Hugo](https://gohugo.io/) was advertised as the fastest possible static site generator. I wanted to use this but I was unfamiliar with golang and didn't really want to learn it just for this.
3. [Jekyll](https://jekyllrb.com/) looked solid as it is used by githubpages, however I have had some bad encounters with ruby so I passed it.

## Chosen platform

I chose hexo probably because it was the first one that I tried. I'm also
familiar with javascript so tweaking is easier.

## Setting up hexo

Hexo is pretty straightforward to get working. Install it globally with
```sh
sudo npm install hexo -g
```
and create a blog with
```sh
hexo init [folder]
```

### Theme

There was a lot of themes out there. I wanted something simple. No fancy animations just usability. I found [Cactus](https://github.com/probberechts/hexo-theme-cactus) and forked it. The code background colors were way too dim on the white variant so I tweaked it a little. Repo can be found [here](https://github.com/eemed/hexo-theme).

### Latex support

To add latex support to the site I used [markdown-it-plus](https://github.com/CHENXCHEN/hexo-renderer-markdown-it-plus) renderer, downloaded [katex](https://katex.org/) font and css and created `_katex/index.styl` file.

```css
@font-face
  font-family: Katex
  src: url('../../lib/katex/KaTeX_Main-Regular.ttf'),
       url('../../lib/katex/KaTeX_Main-Regular.woff2') format('woff2'),
       url('../../lib/katex/KaTeX_Main-Regular.woff') format('woff'),
       url('../../lib/katex/KaTeX_Main-Regular.ttf') format('truetype')
  font-weight: normal

@import url('../../lib/katex/katex.min.css')
```

Then imported it to the main `styles.styl` file

```css
@import "_katex"
```

## Publishing

At this point I thought about the workflow that I would have. Found some hexo documentation on usage with githubpages and travisCI. This seemed to fit me pretty well. I followed [this guide](https://hexo.io/docs/github-pages) with some changes to `.travis.yml`. Githubpages insisted to use the `master` branch so I switched the hexo project to `dev` branch.

```yaml
sudo: false
language: node_js
node_js:
  - 10 # use nodejs v10 LTS
cache: npm
branches:
  only:
    - dev # build master branch only
script:
  - hexo generate # generate static files
deploy:
  provider: pages
  skip-cleanup: true
  github-token: $GH_TOKEN
  keep-history: true
  target_branch: master # <-- New
  on:
    branch: dev
  local-dir: public
```

Now I could just commit the posts to `dev` branch travisCI would pick them up and generate the static pages that will be served by githubpages.

## Troubles

### TravisCI

At first travisCI wouldn't push the pages to the `master` branch. This was solved by creating a dummy branch with nothing in it. Then travis would overwrite everything in that branch Then travis would overwrite everything in `master` branch.

### Git

Cactus theme is a submodule in the github repository. I hadn't dealt with submodules before. For the few first attempts to build the pages they were empty because travisCI didn't clone the theme. After adding the theme as a submodule with
```sh
git submodule add https://github.com/eemed/hexo-theme themes/cactus
```
the pages generated properly.
