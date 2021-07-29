---
title: DIY Static Site Generator in an Afternoon - Part 2
date: 2021-07-29 17:42:02
tags:
- web
- javascript
- static-site-generators
---

So recently I needed to create a very simplistic website. No React, no fancy server-side things, no fuzz. I could probably do this with plain old HTML, CSS and vanilla JS, but I wanted something slightly more sophisticated.

In this second part I will describe how to set up the template generation. Please refer to  {% post_link diy-static-site-generator-pt-1 "part 1" %} for the background and {% post_link diy-static-site-generator-pt-3 "part 3" %} for how to set up the asset build pipeline.

## Tools to the trade

How do we generate HTML files from Markdown and templates? Many static site generators use what they call front matter, usually a Yaml snippet at the beginning of the markdown file. Imo this has two downsides; a) It gets hard to parse the Yaml (because it's actually blended with Markdown), and b) I don't like Yaml (this is for a future blog post though).

My favorite language for configuration is probably [TOML (Tom's Obvious Minimal Language)](https://toml.io/en/) because - as the name suggest - it's obvious and minimal, yet complete for most use cases. So let's use TOML for the content.

As for templating HTML, I like [Pug](https://pugjs.org/api/getting-started.html). It takes away the verbosity and quirks HTML comes with.

## The content and template

With these two content files and the template file we should be able to set up a complete Gulp task that generates HTML files from the TOML and Pug files.

```toml src/content/index.toml
# This file should compile into `dist/index.html` 
template = "index.pug"

title = "Hello"
```

```toml src/content/goodbye.toml
# This file should compile into `dist/goodbye/index.html`
template = "index.pug"

title = "Goodbye"
```

```pug src/templates/index.pug
html
    head
        title #{title}

    body
        h1 #{title}
```

## The gulp task

Firstly let's set up the Gulp file with the job we create. We want to apply this transformer to every `.toml` file in the `src/content` directory.

```js gulpfile.js
import { templates } from './gulp/templates.js'

const templatesDir = path.resolve(process.cwd(), 'src/templates') // Used for the `templates` function to know where to look for `index.pug`
const templatesBaseDir = path.resolve(process.cwd(), 'src/content') // Used for the `templates` function to know where the content base is.

// ...

export const processTemplates = () =>
  gulp
    .src('src/content/**/*.toml') // Apply the `templates` job on all `.toml` files in the `src/content` directory
    .pipe(templates({ baseDir: templatesBaseDir, templatesDir })) // actually run the job (we'll get to this part soon)
    .pipe(gulp.dest('dist')) // Output the resulting HTML files directly in the `dist` directory

export default gulp.series(copyAssets, processSass, processTemplates)
```

For every `.toml` file we will take a few steps:

- parse the TOML contents
- find the corresponding `.pug` template file defined as `template = "my-template.pug"` in the toml file
- compile the Pug template file, execute it and pass the fields defined in the `.toml` file as variables to the template
- Emit a directory containing an `index.html` file, but with the corresponding name of the `.toml` file (for example `my-page.toml` -> `my-page/index.html`). This is to allow browsers to go to i.e. `https://mywebsite.com/my-page/` (pretty url) instead of `https://mywebsite.com/my-page.html` (ugly url).

```js gulp/templates.js
import toml from 'toml'
import pug from 'pug'
import { Transform } from 'stream'
import path from 'path'
import Vinyl from 'vinyl'

export const templates = ({ baseDir, templatesDir }) => {
  const stream = new Transform({ objectMode: true }) // Streams are at the heart of Gulp, helping us to transform all the files

  stream._transform = async function (chunk, _unused, callback) {
    const config = toml.parse(chunk.contents.toString()) // `chunk` represents the .toml file we're transforming into a template
    const templatePath = path.resolve(templatesDir, config.template) // inside the TOML file we specify which template file to use
    let result
    try {
      result = pug.compileFile(templatePath)(config) // By passing the config (TOML file) to the template all configuration keys will be available as variables
    } catch (e) {
      callback(e) // Oops, tell Gulp sometihng went wrong by passing the error to the callback
      return
    }

    let htmlFilePath = chunk.path.replace(/\.toml$/, '.html') // Reuse the template's name as the HTML file name
    if (!htmlFilePath.endsWith('/index.html')) {
      // if the file isn't already named index we want to create a directory with an index file to get the pretty URLs
      htmlFilePath = htmlFilePath.replace(/\.html$/, '/index.html')
    }

    // Vinyl is used to represent virtual files in Gulp. As we're transforming the `.toml` file we need to inform Gulp that now we have a `.html` file instead
    const resultingFile = new Vinyl({
      path: htmlFilePath,
      base: baseDir,
      contents: new Buffer(result),
    })

    callback(null, resultingFile) // Tell Gulp the transformation of this file is done. The `.toml` file is now a `.html` file
  }

  return stream // The `stream` is what'll be passed to `.pipe` 
}
```

Now we should be able to run `yarn gulp processTemplates` (where `processTemplates` refers to the exported function in `gulpfile.js`), and the template files should become `.html` files in the `dist` directory.

To see your (very sparse) website in action you can use the very basic `http-serve` package by installing `yarn -D http-serve` and use `yarn hs dist`. To not have to memorize the build and hs commands lets add this to `package.json`

```json package.json
// ...
  "scripts": {
    "build": "gulp",
    "serve": "hs ./dist"
  }
// ...
```

## Markdown

So far we're just putting regular text in here, but what about the markdown?

It turns out Pug has a markdown filter, but it's applied _before_ variables, meaning we cannot use it as we want to write markdown in the TOML config files. This means we have to take care of the markdown parsing in our templates renderer.

We'll amend with the following code:

```toml src/content/index.toml src/content/goodbye.toml
# This file should compile into `dist/index.html`
template = "index.pug"

title = "Hello" # And, for now, Goodbye for goodbye.toml
headline = "inlinemd:A beautiful headline with _italic_ content" ## We don't want markdown to automatically add `<p>` tags here
content = """md:
## A sub headline

__Very__ informative text
""" # But we want `<p>` tags here
```

```pug src/templates/index.pug
html
    head
        title #{title}

    body
        // `!` instead of `#` means we bypass the escaping of html characters
        h1 !{headline}

        .
            !{content}
```

```js gulp/templates.js
import marked from 'marked'

// ...

stream._transform = async function (chunk, _unused, callback) {
  const config = toml.parse(chunk.contents.toString()) // `chunk` represents the .toml file we're transforming into a template
  const templatePath = path.resolve(templatesDir, config.template) // inside the TOML file we specify which template file to use

  let result
  try {
    const markdownedConfig = Object.entries(config) // Loop over every configuration key, as we want to check whether it should be parsed as markdown.
      .map(([key, value]) => {
        // If we prefix a string config value with `md:` that means we want to parse it as markdown
        if (typeof value === 'string' && value.startsWith('md:')) {
          return [key, marked(value.replace('md:', ''), {})] // remove the `md:` prefix and parse the rest as markdown. `.replace` will only replace the first occurrence so no worries if we happen to include `md:` in the actual content
        } else if (
          typeof value === 'string' &&
          // If we prefix the string config value with `inlinemd:` that means we want to parse it as inline markdown
          value.startsWith('inlinemd:')
        ) {
          return [key, marked.parseInline(value.replace('inlinemd:', ''), {})] // same thing as the `md:` prefix, but do it inline
        } else {
          // This should not be parsed as markdown
          return [key, value]
        }
      })
      .reduce((acc, [key, value]) => ({ ...acc, [key]: value }), {}) // Put together the configuration as it was before, but with the parsed markdown

    result = pug.compileFile(templatePath)(markdownedConfig)
  } catch (e) {
    callback(e)
    return
  }
  
  // ...

}

// ...
```

As we let the `marked` package take care of markdown parsing, install it with `yarn add -D marked`.

Now, running `yarn build` and `yarn hs` should show you something like this:

![screenshot](screenshot.png)

For a fully working example of everything stitched together, see the git repo [palmenhq/static-site-generator#part-2](https://github.com/palmenhq/static-site-generator/tree/part-2).

Continue to {% post_link diy-static-site-generator-pt-3 "part 3" %} for how to build the asset pipeline.
