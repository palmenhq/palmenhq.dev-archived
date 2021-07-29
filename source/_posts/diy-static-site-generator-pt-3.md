---
title: DIY Static Site Generator in an Afternoon - Part 3
date: 2021-07-29 18:32:02
tags:
- web
- javascript
- static-site-generators
---

So recently I needed to create a very simplistic website. No React, no fancy server-side things, no fuzz. I could probably do this with plain old HTML, CSS and vanilla JS, but I wanted something slightly more sophisticated.

In this third part I will describe how to set up the asset build pipeline. Please refer to  {% post_link diy-static-site-generator-pt-1 "part 1" %} for the background and {% post_link diy-static-site-generator-pt-2 "part 2" %} for how to set up the templates generation.

## Tools to the trade

We already set up Sass, but we'll amend it with some improvements. Also we'll use Webpack (yup!) to bundle and process our Javascript. Additionally we will set up live reload (no hot reolading in this project, sorry :/) and watching, so you don't have to run `yarn build` and update the page manually on every file change to see the result.

There are two hard things in computer science - naming (which I will not cover in this section) and cache invalidation. What the latter means, is that whenever we make a new release of our website, we want the browser's cache to be updated - meaning we don't serve the old content. "Can we not turn off caching to get rid of this problem?" you wonder. The simple answer is that we can, but it will make our website way less efficient than it should be - downloading the same content over and over. 

### Cache busting for the styling

We already set up the styling in {% post_link "diy-static-site-generator-pt-1" "part 1" %}, but we need a mechanism that busts (invalidates) the cache for us when we change the content. It took me a while to come up with this solution, but I think it works.

A common way to bust the cache is to add a content hash to the file name (so the `index.css` file will be renamed to something like `index-7025eefd346ef6f3abad8d51e5d43ffa.css`). What the random string does is give a new file name every release where the content is updated, forcing the browsers to download the new file only when the content is changed. To implement this, we need to first a) generate the hash for the file and b) somehow pass that file to the template so we can import it in our HTML.

#### Content hashing

Turns out gulp has this nifty plugin called `gulp-hash-filename` - just what we need! We only want this for production builds, so additionally we'll install `gulp-mode` to let us differentiate dev builds from prod builds: `yarn add -D gulp-hash-filename gulp-mode` 

Then amend our gulpfile:

```js gulpfile.js
import hash from 'gulp-hash-filename'
import gulpMode from 'gulp-mode'

// ...

export const processSass = () =>
  gulp
    .src('src/sass/index.scss') // index.scss is our entrypoint
    .pipe(sass().on('error', sass.logError)) // Let Sass work its magic 🪄
    
    .pipe(mode.production(hash())) // Hash our content, but only when making a production build
    
    .pipe(gulp.dest('dist/css')) // Output directory is `dist/css`
```

If you run `NODE_ENV=production yarn build` (`NODE_ENV` lets us select `mode`), instead of getting a `index.css` in `dist/css`, you should see something like `index-7025eefd346ef6f3abad8d51e5d43ffa.css`.

Now we have new file names when the content changes, but how do we get this, seemingly random, filename in to our content? It'll change with its contents, leaving us no way of predicting it. This was the tricky part, but I ended up writing a little custom function that picks up all content hashes, letting us inject it into our templates:

```js gulp/export-hash.js
import { Transform } from 'stream'

export const fileRegistry = new Map() // This is where we'll fetch the file names from

export const exportHash = () => {
  const stream = new Transform({ objectMode: true }) // Another stream

  stream._transform = function (chunk, _unused, callback) {
    const originalName = chunk.history[0].replace(chunk.base, '') // Find the original name the file had, it'll be the first entry in the file's `history` property
    const newName = chunk.history[chunk.history.length - 1].replace(
      chunk.base,
      ''
    ) // Remove the base path (ie `src/sass`), I think the content hashed files should be few enough to not have colliding file names

    fileRegistry.set(originalName, newName) // Store away the file name as an entry `index.css -> index-7025eefd346ef6f3abad8d51e5d43ffa.css`

    this.push(chunk) // Just continue, letting gulp know we finished
    callback()
  }

  return stream
}
```

We can now amend our gulpfile:

```js gulpfile.js
import { exportHash } from './gulp/export-hash.js'

// ...
export const processSass = () =>
  gulp
      // ...
      .pipe(mode.production(hash())) // Hash our content, but only when making a production build
      .pipe(mode.production(exportHash())) // Our brand new function to keep track of hashed file names
```

So how do we get this into our template? Easy!

```pug src/templates/index.pug
html
    head
        // `fileRegistry` comes from `gulp/export-hash.js`. We try to get `/index.scss`, and
        // if that isn't present probably we're in dev mode (with no content hashing), and we can output the regular file name
        link(rel="stylesheet", href=('/css' + (fileRegistry.get('/index.scss') || '/index.css')))

        title #{title}

    body
        // `!` instead of `#` means we bypass the escaping of html characters
        h1 !{headline}

        .
            !{content}
```

But where does the `fileRegistry` variable come from now? We do, of course, need some glue to connect `fileRegistry` to our templates:

```js gulpfile/templates.js
import { fileRegistry } from './export-hash.js'

// ..

export const templates = ({ baseDir, templatesDir }) => {
  // ...

  try {
 
    // ...

    result = pug.compileFile(templatePath)({
      ...markdownedConfig,
      fileRegistry, // add the fileRegistry as a variable in the template
    })
  } catch (e) {
    // ...
  }
  
  // ...
}
// ...
```

Now, your website should look something like the following:

![](screenshot.png)

Ah, what a beautiful website.

## CSS post-processing

In order to improve the developer experience, be more backwards compatible with older browsers, and decrease loading times there's a few plugins we want to apply to the CSS/Sass build pipeline: `yarn add -D gulp-autoprefixer gulp-sourcemaps gulp-cssnano`.

- autoprefixer - Adds, well, prefixes to our CSS - for example it adds `-moz-box-sizing` if we add the `box-sizing` property somewhere
- sourcemaps - Adds source maps to our code, so that we can inspect the CSS in our browser and see which Sass file it comes from
- cssnano - Minifies the CSS, to decrease the file size

And we'll amend the gulpfile like this:

```js gulpfile.js
import autoprefixer from 'gulp-autoprefixer'
import cssnano from 'gulp-cssnano'
import sourcemaps from 'gulp-sourcemaps'

// ...

export const processSass = () =>
  gulp
    .src('src/sass/index.scss') // index.scss is our entrypoint
    .pipe(mode.development(sourcemaps.init()))
    .pipe(sass().on('error', sass.logError)) // Let Sass work its magic 🪄
    .pipe(
      autoprefixer({
        overrideBrowserslist: ['> 1%'],
      })
    )
    .pipe(cssnano())
    .pipe(mode.development(sourcemaps.write()))
    .pipe(mode.production(hash())) // Hash our content, but only when making a production build
    .pipe(mode.production(exportHash())) // Our brand new function to keep track of hashed file names
    .pipe(gulp.dest('dist/css')) // Output directory is `dist/css`
```

## JavaScript processing and bundling

For the same reasons we apply post-processing to our CSS we'll want to do that to our JS. Also Sass takes care of bundling our scss files into one fat CSS file, whereas for the JS we'll need to take care of that ourselves. Hence we'll add the followig plugins; `yarn add -D gulp-webpack babel-loader @babel/core @babel/preset-env`

- Webpack - Dewat?? Webpack? Yes, we're using Webpack inside Gulp, and we let it do what it does best - take many small files and put them into fat file. We'll also let webpack do the work of transforming and minifying JS files.
- Babel - For the same reason we want autoprefixer, we let babel take care of transforming our modern JS to old-school JS so older browsers understand it

```js gulpfile.js
import babel from 'gulp-babel'
import webpack from 'gulp-webpack'

// ...

export const processJs = () =>
  gulp
    .src('src/js/index.js')
    .pipe(
      webpack({
        mode: mode.development() ? 'development' : 'production', // What to optimize for, ie whether to minify the code or not
        module: {
          rules: [
            // Rules let us define custom file transformers, for example babel
            {
              test: /\.js$/,
              exclude: /(node_modules)/,
              use: {
                loader: 'babel-loader', // Babel lets us use modern JS for older browsers
                options: {
                  presets: ['@babel/preset-env'],
                },
              },
            },
          ],
        },
        output: {
          filename: 'bundle.js',
        },
        devtool: mode.development() ? 'source-map' : 'none', // Instead of using `gulp-sourcemaps` here we want Webpack to take care of our source maps, as that's the plugin that will include all other JS files
      })
    )
    .pipe(mode.production(hash()))
    .pipe(exportHash())
    .pipe(gulp.dest('dist/js'))

// We'll also want to add this to the default task
export default gulp.series(copyAssets, processSass, processJs, processTemplates)
```

For now, lets just add a dummy JS file so the build doesn't fail...

```js src/js/index.js
console.log('it works!')
```

...and import the output file (bundle.js) in our template:

```pug src/templates/index.pug
// ...
    body
        // ...

        script(src='/js' + (fileRegistry.get('/bundle.js') || '/bundle.js'))
```

Lets test the new setup: `yarn build && yarn serve` - when opening our browser we should see something like the following:

![](./screenshot-2.png)

Note the two circled file names - our CSS output file is actually called `index.css` (not `index.scss`), and the output JS file is called `bundle.js` (not `index.js`), meaning the source maps work as they should.

## Watch & live reload

Finally, it'll be quite annoying to have to run `yarn build` and then reload our page for every change we make to our code no? There's a solution to that problem: gulp watch mode and live reload. Firstly, we'll add the following package: `yarn add -D gulp-livereload`

Then we'll amend the `gulpfile.js` with this places:

```js gulpfile.js
export const processJs = ({ watch = false }) => () => // Note that we're wrapping our funcion in a function that takes `{ watch: true || false }` as an arg
  gulp
    .src('src/js/index.js')
    // ...
    .pipe(mode.development(liveReload()))
    .pipe(gulp.dest('dist/js'))

export const processSass = () =>
  gulp
    .src('src/scss/index.scss')
    // ...
    .pipe(mode.development(liveReload()))
    .pipe(gulp.dest('dist/css')) // Output directory is `dist/css`

export const processTemplates = () =>
  gulp
    .src('src/content/**/*.toml')
    // ...
    .pipe(liveReload()) // No need to do this only for development, because here it dosen't affect the build
    .pipe(gulp.dest('dist'))

export const watch = () => {
  gulp.watch(
    ['src/js/**/*.js'],
    { ignoreInitial: false },
    processJs({ watch: true }),
  )
  gulp.watch(['src/scss/**/*.scss'], { ignoreInitial: false }, processSass)
  gulp.watch(['assets/**/*'], { ignoreInitial: false }, copyAssets)

  gulp.watch(
    ['src/content/**/*.toml', 'src/templates/**/*.pug'],
    { ignoreInitial: false },
    processTemplates,
  )
  liveReload.listen()
}
```

Finally, you will need to install a live reload browser plugin for [Firefox](https://addons.mozilla.org/en-US/firefox/addon/livereload-web-extension/) or [Chrome](https://chrome.google.com/webstore/detail/livereload/jnihajbhpnppcggbcgedagnkighmdlei?hl=en). Why, do you ask? It'll listen to the live reload server, which signals to the browser extension whenever a file changed, and the browser extension will automatically reload the page. It's like the cheap version of [hot module replacement](https://stackoverflow.com/questions/24581873/what-exactly-is-hot-module-replacement-in-webpack).

As a final step, gulp watch will build our files on every file change. However, we also need to concurrently serve the `dist` directory to see our files in the browser. We'll use the `concurrently` package for this: `yarn add -D concurrently`. Now we can run `NODE_ENV=development yarn concurrently -n hs,gulp 'yarn hs dist' 'yarn gulp watch'` to develop very neatly. As for the production build, we'll need to run it like `NODE_ENV=production yarn gulp`. However, these are some long commands to remember, so lets ad them to `package.json`:

```json package.json
  "scripts": {
    "build": "NODE_ENV=production gulp",
    "start": "NODE_ENV=development concurrently -n hs,gulp 'yarn hs dist' 'yarn gulp watch'",
    "serve": "hs dist"
  },
```

Now, to develop, you run `yarn start`. When you're ready to deploy your site you run `yarn build`. To test your site after building for production, you can run `yarn serve` to serve the built `dist` directory.

## Final words

Voilà! We have now built a static site generator, and framework for asset processing! The full working example can be found at [palmenhq/static-site-generator#part-3](https://github.com/palmenhq/static-site-generator/tree/part-3) (which you can also compare against if something didn't work after following this tutorial).

The point of this is of course not to spread the [not-invented-here syndrome](https://en.wikipedia.org/wiki/Not_invented_here), but rather prove that these fancy frameworks don't use sorcery - just some semi-advanced configuration. But over all, its not really that hard to create your own static site generator - as long as you know which components to glue together. 

If you made it this far, thank you for reading. Feel free to drop me a line what you thought about this on [Twitter](https://twitter.com/palmenhq)  (or anywhere else is suitable), all feedback is welcome!
