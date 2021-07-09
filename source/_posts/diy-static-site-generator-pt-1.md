---
title: DIY Static Site Generator in an Afternoon - Part 1
date: 2021-07-09 06:42:02
tags:
- web
- javascript
- static-site-generators
---

So recently I needed to create a very simplistic website. No React, no fancy server-side things, no fuzz. I could probably do this with plain old HTML, CSS and vanilla JS. However, it would be nice to be able to pull in a JS lib or two, utilize some Sass features etc. Also, I'm not really a fan of writing HTML content manually (I prefer Markdown), so I started browsing for static site generators that could fit this use case. Gatsby is ruled out, too complex. Hexo (that this site is built on) is too focused on blogs. Jerkyll seems nice but it seems to me it doesn't really take care of the asset build pipeline and so on. There's a million out there already, but I like coding over browsing the web, so I figured - "how hard can it be to do it yourself?". At least it makes a good blog post.

So with three requirements - build Sass to a CSS bundle, transpile ESNext to a bundle and put Markdown in template files - I got started.

## So what is a static site anyways?

At its core, a static site is basically a website that doesn't have any moving parts on the server side. It's just plain HTML, CSS and JavaScript files - much like you built websites in the 90's and 00's. It's a bit of a backlash against the complexity of computing websites on the server with for example PHP or Node, as it often decreases complexity and latency during page load (as you're only serving files).

This works very well for Single Page Applications (SPAs), where it's ok (sometimes even preferred) for dynamic content to be loaded client-side. But when you want dynamic content (for example copy and images from a CMS) to be included in the HTML sent from your server this becomes impossible to do "just in time" when you serve the content from static files. This is where static site generators come in as they can fill templates with dynamic content during a build step, ultimately generating static HTML files, instead of during runtime. This lets you detect potential errors before the content is served and decreases latency as you only do the "work" once instead of at every request.

Commonly static site generators are really frameworks that help you with the build pipeline, including for example a JavaScript bundler (ie Webpack) and CSS preprocessor (ie Sass), making the development experience easier.

## Tools to the trade

When you say "I need a task runner", many would probably respond "Webpack" on auto-pilot. Webpack is more of a bundler than a task runner though - the difference being that the purpose is to take multiple files and make on fat file from them, while a task runner's purpose is to - well - help you automate and run any kind of task.

Looking at actual task runners, it seems [Grunt](https://gruntjs.com/) and [Gulp](https://gulpjs.com/) are still the two main options. I decided to go with Gulp, as I figured their API is a bit more straightforward compared to Grunt. It seems Gulp is a little stale (the latest release at the time of writing was made in 2019), but it still does the job.

### Gulp crash course

#### Installation

As I prefer Yarn for package management I will use `yarn add -D gulp` to install it.

#### Usage

At its core Gulp builds on having input files and processing them as a stream, meaning you take a file and apply different plugins on it. For example, to create a task that copies files from one directory to another without processing them in any way, you place this inside a file named `gulpfile.js`. Then run `yarn gulp` to run the task.

```js gulpfile.js
export const copyAssets = () =>
  gulp.src('assets/**/*') // Take any file inside the `assets` directory...
    .pipe(gulp.dest('dist/assets')) // ...and copy it to `dist/assets`

export default gulp.series(copyAssets) // Run `copyAssets` when you run `yarn gulp`
```

To take a Sass file and let Sass produce a CSS file from it, install the sass packages with `yarn add -D node-sass gulp-sass`, place this in your `gulpile.js` and run `yarn gulp`.

```js gulpfile.js
import gulpSass from 'gulp-sass'
import nodeSass from 'node-sass'

const sass = gulpSass(nodeSass)

// ...

export const processSass = () =>
  gulp
    .src('src/scss/index.scss') // index.scss is our entrypoint
    .pipe(sass().on('error', sass.logError)) // Let Sass work its magic 🪄
    .pipe(gulp.dest('dist/css')) // Output directory is `dist/css`

export default gulp.series(copyAssets, processSass) // Run `copyAssets` and `processSass` when you run `yarn gulp`
```

```scss src/scss/index.scss
body {
  background: pink;
  color: red;
  font-family: "Comic Sans MS", sans-serif;
}
```

Please note that in order to use the ESModules (`import` statements) you'll need to put `"type": "module"` in `package.json`.

A working example of the above code can be found on GitHub: [palmenhq/static-site-generator#part-1](https://github.com/palmenhq/static-site-generator/tree/part-1)

Continue to {% post_link diy-static-site-generator-pt-2 "part 2" %} for how to build the templates and {% post_link diy-static-site-generator-pt-3 "part 3" %} for how to build the asset pipeline.