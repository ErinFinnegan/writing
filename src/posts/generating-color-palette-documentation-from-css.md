---
title: Generating Color Palette Documentation from CSS
subhead: How to use Node.js modules to create color documentation for living style guides
created: 06-12-2015
tags:
  - css
  - color
  - documentation
  - style guide
  - living style guide
  - color contrast
  - accessibility
  - a11y
related:
  - name: Colorable
    href: http://jxnblk.com/colorable
  - name: postcss
    href: https://github.com/postcss/postcss
  - name: Basscss Color Combinations
    href: http://basscss.com/docs/reference/color-combinations
draft: true
---

Virtually every style guide has a color palette section in its documentation.
Many times I’ve seen this documentation created manually, where every change to a color requires updating the values in two places – the stylesheet and the style guide.
This often leads to the style guide falling out of sync with the actual values used on a site,
and makes maintaining a living style guide less efficient.

The problem with this approach is that two different things are serving as a source of truth.
For a true living style guide, the actual code should be the definitive source of truth,
and color values can be extracted from the CSS.
Doing this can help expose legacy colors that should be removed
and can point out ways to iteratively refactor and DRY up a code base.

<!-- so why not use CSS to populate the data for documentation? -->

The following will show how to use a combination of Node.js modules to build basic data-driven documentation for color palettes as well as present color contrast values for each combination as a guide for usage.

## Initial Setup

If you’re already using Node.js in your build system,
you can integrate this directly into your project.
For the purposes of this tutorial, create a new sandboxed project to experiment with.

```bash
mkdir color-documentation
cd color-documentation
npm init
```

Install the following modules.

```bash
npm i --save-dev postcss color colorable lodash
```

The [postcss](https://github.com/postcss/postcss) module will be used to transform the source CSS into an
<a href="http://en.wikipedia.org/wiki/Abstract_syntax_tree"><abbr title="Abstract Syntax Tree">AST</abbr></a>
for manipulation with JavaScript.
[Color](https://www.npmjs.com/package/color) will be used to convert the values found in the stylesheet to hexidecimal.
[Colorable](http://jxnblk.com/colorable) will be used to get color contrast values and
[WCAG](http://www.w3.org/TR/WCAG20/#visual-audio-contrast)
scores for every possible combination of colors.
And [lodash](https://lodash.com/docs) will be used to find unique color values and for creating a page template.

Copy a CSS file into the project to use as a source for the color values.
For this tutorial, you can use any stylesheet with colors or just download [Basscss](http://www.basscss.com/docs/).

## Build Script

Create a new `build.js` file that will be used to read the CSS file and eventually to generate an HTML page.

```js
// build.js
var fs = require('fs')

// Read the contents of basscss.css
var css = fs.readFileSync('basscss.css', 'utf8')

console.log(css)
```

For now, this script just prints the stylesheet.

## Package Scripts

Add a script to `package.json` for the build script.

```json
// package.json
"scripts": {
  "start": "node build"
}
```

Run the script to make sure everything is working.

```bash
npm start
```

## Parsing CSS

Create a new `lib/parse-css.js` for parsing the CSS.

```js
// lib/parse-css.js
var _ = require('lodash')
var postcss = require('postcss')
var Color = require('color')

module.exports = function(css) {
  // Array of colors to return
  var colors = []

  // Parse the CSS file and get the AST
  var root = postcss.parse(css)

  // Iterate through every declaration and log the property
  root.eachDecl(function(decl) {
    console.log(decl.prop)
  })

  return colors
}
```

Use this module in `build.js`.

```js
// build.js
var fs = require('fs')
var parseColors = require('./lib/parse-colors')

// Read the contents of basscss.css
var css = fs.readFileSync('basscss.css', 'utf8')
var colors = parseColors(css)
```

The `parse-colors` module takes raw CSS and transforms it into an AST that can be iterated over.
For now, this just logs all the properties found in the stylesheet.

## Filtering Colors

Next update the parse module to filter out everything but color and background-color properties,
convert the values to hex, and remove duplicates.
While color values can be used in other properties including the `background` shorthand,
handling that would add complexity and will be omitted from this tutorial.

```js
// lib/parse-colors.js
var _ = require('lodash')
var postcss = require('postcss')
var Color = require('color')

module.exports = function(css) {

  var colors = []
  var root = postcss.parse(css)

  // Iterate through every color and background-color declaration
  root.eachDecl(/(color|background\-color)/, function(decl) {
    // Add each color to the colors array
    colors.push(decl.value)
  })

  // Convert all color values to hex strings
  colors = colors.map(function(color) {
    var hex
    try {
      hex = Color(color).hexString()
      return hex
    } catch(e) {
      // Handle color values like inherit and currentcolor
      return false
    }
  })
  .filter(function(value) {
    // Remove false values
    return value
  })

  // Remove duplicate values
  colors = _.uniq(colors)

  return colors

}
```

The first argument in `root.eachDecl` is a regular expression to filter declarations for either `color` or `background-color`.
See the [postcss documentation](https://github.com/postcss/postcss/blob/master/docs/api.md#containereachdeclpropfilter-callback) for more details.
The `Color().hexString()` method converts any valid color value to hex format. The lodash `_.uniq` method removes duplicate values from an array.

## HTML Template

Next, create a `template.html` file for displaying the colors.
This tutorial uses lodash templates, but any JavaScript templating language could work.
Note: if you’re using a stylesheet other than Basscss, the classes applied below may differ.

```html
<!DOCTYPE html>
<!-- template.html -->
<html>
<head>
  <meta charset="utf-8">
  <title>Color Palette Docs Demo</title>
  <link rel="stylesheet" href="basscss.css">
</head>
<body class="px3">
  <header class="py3">
    <h1>Color Palette Docs Demo</h1>
  </header>
  <ul class="list-reset flex flex-wrap flex-justify mxn2">
    <% colors.map(renderColor) %>
  </ul>
  <% function renderColor(color) { %>
    <li class="m2">
      <div style="background-color:<%= color %>"
        class="p4"></div>
      <%= color %>
    </li>
  <% } %>
</body>
</html>
```

Inside the `<ul>` the `.map()` method is used to iterate over the `colors` array and passes the `renderColor` function as a callback.
The `renderColor` function then renders an `<li>` for each color with a div that has its background color set to that value.

Edit `build.js` to read the template and generate an HTML file.

```js
// build.js
var _ = require('lodash')
var fs = require('fs')
var parseColors = require('./lib/parse-colors')

var css = fs.readFileSync('basscss.css', 'utf8')
// Read the template string
var template = fs.readFileSync('template.html', 'utf8')
// Create a lodash template function
var tpl = _.template(template)
var colors = parseColors(css)

// Render the template function to an HTML string
var html = tpl({ colors: colors })

// Write an HTML file to disk
fs.writeFileSync('index.html', html)
```

Now run `npm start` and the build script should generate an HTML file with the colors parsed from the stylesheet.

```bash
npm start && open index.html
```

<!-- img of color tiles -->

## Readable Color Combinations

The Colorable module takes an array of colors and returns a nested array of color combinations, along with their contrast values to test for readability. This can be useful for seeing what foreground-background pairs can and cannot be used.

Create a new `lib/parse-combos.js` file.

```js
// lib/parse-combos.js
var colorable = require('colorable')

module.exports = function(colors) {

  var combos = []
  // Get the Colorable array
  var arr = colorable(colors)

  // Flatten the array and combine the foreground and background colors
  arr.forEach(function(color) {
    var primary = color.hex
    color.combinations.forEach(function(combo) {
      combo.primary = primary
      combos.push(combo)
    })
  })

  // Sort the array by contrast from high to low
  combos.sort(function(a, b) {
    return b.contrast - a.contrast
  })

  return combos

}
```

Create the combos array in `build.js` and pass the combos array into the template function.

```js
// build.js 
var _ = require('lodash')
var fs = require('fs')
var parseColors = require('./lib/parse-colors')
var parseCombos = require('./lib/parse-combos')

var css = fs.readFileSync('basscss.css', 'utf8')
var template = fs.readFileSync('template.html', 'utf8')
var tpl = _.template(template)
var colors = parseColors(css)
var combos = parseCombos(colors)

var html = tpl({
  colors: colors,
  combos: combos
})
fs.writeFileSync('index.html', html)
```

Add a section to display the color combinations in `template.html`.

```html
  <h2>Combinations</h2>
  <ul class="list-reset flex flex-wrap">
    <% combos.map(renderCombo) %>
  </ul>
  <% function renderCombo(combo) { %>
    <li class="py2 col-6 sm-col-4 md-col-3 lg-col-2"
      style="color:<%= combo.primary %>;background-color:<%= combo.hex %>">
      <div class="h1 bold px2">
        Aa
      </div>
      <div class="h5 px2">
        <%= combo.primary %>
        <br>
        <%= combo.hex %>
        <br>
        <%= combo.contrast.toFixed(2) %>
      </div>
    </li>
  <% } %>
```

Run the build script. You should now have a list of color combinations along with the contrast value for each pair.

<!-- image of color combinations -->

While seeing combinations that don’t have enough contrast might be useful,
for this tutorial set the threshold to 3 in Colorable’s options
to only show combinations that pass the WCAG minimum for large text.
Edit the options for Colorable in the `parse-combos` module.

```js
// lib/parse-combos.js
  // ...
  var combos = []
  var arr = colorable(colors, { threshold: 3 })
  // ...
```

Run the build script again. Now you should only see color combinations with a contrast value of 3 or above.

At this point, feel free to edit the styles of the rendered template and explore different ways of showing this information. You can even swap `basscss.css` out for another stylesheet to see how this looks in other frameworks.

## Expanding on This Idea

Using postcss, things like the selectors used for each color
or the number of times each color is used in a stylesheet can be incorporated into this approach.
You could also compare how similar colors are to help expose
inconsistencies and opportunities to normalize the design.

## Wrap Up

While this is not a complete replacement for human written documentation,
and creating guidelines around color usage requires carefully considered writing,
using an automated tool to generate data-driven documentation
for things like color values can help create a better living style guide.

