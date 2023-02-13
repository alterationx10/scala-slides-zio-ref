# scala-slides

This is a template repository set up to make Scala-centric slide-decks using
`mdoc` and `marp`.

## Prerequisites

You will need to have `scala-cli` installed (manually), and a `node` environment
(which will manage `marp` and the build script).

See the [official site](https://scala-cli.virtuslab.org/) to find your best way
to install `scala-cli`.

## How it works

All the files are in place to start working on your presentation! This expects
your presentation to be `slides.md`, and our Scala script to be `slides.mc`. All
you (hopefully) need to do is start editing `slides.md`!

## Building

You will need to `yarn install` to install `marp`, and then you can run
`yarn build` which will process the files to build your slide deck. It will be
output to `build/index.html`.

## Deploying

There is an included GitHub Action file to build/deploy to GitHub pages on push
to `main`! Make sure your repo is set up to deploy Pages from GitHub Actions!
