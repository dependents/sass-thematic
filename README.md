![SassThematic](title.png)

**A framework for generating dynamic theme stylesheets from Sass.**

* [How it works](#how-it-works)
* [Installation, Upgrade, and API](#install)
* [Full API options](#full-api-options)
* [Webpack integration](#webpack-plugin)
* [Gulp integration](#gulp-pipe)
* [Credits](#credit)

## The Problem:

We're building a site that gets themed with customizable colors, fonts, sizes, etc. So, we set up a base stylesheet for the site, and then maintain a separate theme stylesheet for custom style overrides.

This works, but makes updates difficult. All changes in the base stylesheet must be mirrored in the theme-specific overrides. Keeping these stylesheets synchronized is tedious and error-prone. It would be great if we could just _automate_ the generation of these theme overrides from the base source...

This is SassThematic.

## How it works:

### 1. Structure

First, we setup a sanctioned file in our Sass library that defines all theme variables. We'll extract the names of these theme variables when generating a theme stylesheet. For example:

**In *_theme_vars.scss*:**

```css
$theme-color: green;
```

**In *_other_vars.scss*:**

```css
$other-color: red;
```

**In *main.scss*:**

```css
@import 'theme_vars';
@import 'other_vars';

@mixin mixin-other {
  color: $other-color;
  font-family: serif;
}

@mixin mixin-theme {
  color: $theme-color;
  font-family: serif;
}

.other {
  color: $other-color;
  font-family: serif;
}

.theme {
  color: $theme-color;
  font-family: serif;
}

.include-other {
  @include mixin-other;
}

.include-theme {
  @include mixin-theme;
}
```

Now we can run SassThematic with references to our theme variables file, and to our main Sass file. All SassThematic methods operate similar to [node-sass](https://www.npmjs.com/package/node-sass), with an `includePaths` option for resolving imports:

```javascript
var sassThematic = require('sass-thematic');

sassThematic.parseThemeSass({
  varsFile: './styles/_vars.scss',
  file: './styles/main.scss',
  includePaths: ['./lib/']
},
function(err, sassString) {
  console.log(sassString);
});
```

### 2. Parse

Next, SassThematic reconstitutes our main Sass file's deeply-nested source tree of `@import` statements, and then parses that flattened source into a complete abstract syntax tree (AST) using the fabulous [gonzales-pe](https://github.com/tonyganch/gonzales-pe) lexer:

```css
$theme-color: green;
$other-color: red;

@mixin mixin-other {
  color: $other-color;
  font-family: serif;
}

@mixin mixin-theme {
  color: $theme-color;
  font-family: serif;
}

.other {
  color: $other-color;
  font-family: serif;
}

.theme {
  color: $theme-color;
  font-family: serif;
}

.include-other {
  @include mixin-other;
}

.include-theme {
  @include mixin-theme;
}
```

### 3. Prune

Now SassThematic traverses the parsed AST, dropping any rulesets and/or declarations that do not implement a theme variable (dropped syntax is replaced by a comment). This pruning accounts for `@include`, `@extend`, and many other inflected rule dependencies. This results in a minimal Sass file that can be compiled with new theme variables prepended:

```css
// varsfile
$other-color: red;

// mixin

@mixin mixin-theme {
  color: $theme-color;
  // declaration
}

// ruleset

.theme {
  color: $theme-color;
  // declaration
}

// ruleset

.include-theme {
  @include mixin-theme;
}
```

### 4. Template

Parsing theme variables into a view template is generally simpler to integrate than compiling custom assets for each theme. Therefore, we can also render our Sass theme into flat CSS with variable names passed through as template fields:

```css
.theme { color: <%= theme-color %>; }
.include-theme { color: <%= theme-color %>; }
```

The only caveat with generating templates is that variable names need to pass through the actual Sass compiler as _literals_, therefore we cannot use theme variables as function arguments (ie: `tint($this-will-explode, 10)`) or in math expressions (ie: `$sad-trombone * 0.5`).

## Install

Install the NPM package:

```
npm install sass-thematic --save-dev
```

## Upgrading to v1.x

While the v1.x framework has the same API as the v0.x series, internal operations of the tool have changed significantly, thus meriting the major version bump. Potentially breaking changes:

 - Dropped support for directory imports. Importing follows official [Sass import docs](http://sass-lang.com/documentation/file.SASS_REFERENCE.html#import).
 - Sass version to v3.3 for rendering methods. Dropped support for Sass v2.x.

## API

SassThematic provides the following API. All methods take roughly the same options, which are fully [outlined below](#full-api-options). As of v1.x, all methods have sync and async implementations.

### parseAST
- **sassThematic.parseAST( options, callback )**
- **sassThematic.parseASTSync( options )**

Parses and returns a raw abstract syntax tree of your deeply-nested Sass source. The returned object is a [gonzales-pe](https://github.com/tonyganch/gonzales-pe) node tree with all `@import` statements replaced by the imported stylesheet nodes. Use this complete source tree to make your own modifications.

```javascript
var sassThematic = require('sass-thematic');

// Async
sassThematic.parseAST({
  file: './styles/main.scss',
  includePaths: ['./lib/']
}, function(err, ast) {
   console.log(ast);
});

// Sync
var ast = sassThematic.parseASTSync({ ...options... });
```

### parseThemeAST
- **sassThematic.parseThemeAST( options, callback )**
- **sassThematic.parseThemeASTSync( options )**

Parses, prunes, and returns an abstract syntax tree of just your Sass that implements theme variables. A `varsFile` option is required to identify relevant theme variables. This variables file should include *nothing* but variable definitions. The returned object is a [gonzales-pe](https://github.com/tonyganch/gonzales-pe) node tree.

```javascript
var sassThematic = require('sass-thematic');

// Async
sassThematic.parseThemeAST({
  file: './styles/main.scss',
  varsFile: './styles/_theme.scss',
  includePaths: ['./lib/']
}, function(err, ast) {
   console.log(ast);
});

// Sync
var ast = sassThematic.parseThemeASTSync({ ...options... });
```

### parseThemeSass
- **sassThematic.parseThemeSass( options, callback )**
- **sassThematic.parseThemeSassSync( options )**

Parses, prunes, and returns a rendered Sass string of rules that implement your theme variables. A `varsFile` option is required to identify relevant theme variables. The returned string is raw Sass with all theme variable imports removed. You may prepend new theme variable definitions onto this Sass string and run it through the Sass compiler.

```javascript
var sassThematic = require('sass-thematic');

// Async
sassThematic.parseThemeSass({
  file: './styles/main.scss',
  varsFile: './styles/_theme.scss',
  includePaths: ['./lib/']
}, function(err, sassString) {
   console.log(sassString);
});

// Sync
var sassString = sassThematic.parseThemeSassSync({ ...options... });
```

### renderThemeCSS
- **sassThematic.renderThemeCSS( options, callback )**
- **sassThematic.renderThemeCSSSync( options )**

Parses, prunes, compiles, and returns a rendered CSS string of selectors that implement your theme variables. A `varsFile` option is required to identify relevant theme variables. A `themeFile` or `themeData` option is required to provide variables used to render the CSS.

```javascript
var sassThematic = require('sass-thematic');

// Async
sassThematic.renderThemeCSS({
  file: './styles/main.scss',
  varsFile: './styles/_theme.scss',
  themeData: '$color1: red; $color2: green;',
  includePaths: ['./lib/']
}, function(err, cssString) {
   console.log(cssString);
});

// Sync
var cssString = sassThematic.renderThemeCSSSync({ ...options... });
```

### renderThemeTemplate
- **sassThematic.renderThemeTemplate( options, callback )**
- **sassThematic.renderThemeTemplateSync( options )**

Parses, prunes, compiles, and returns a rendered template string of flat CSS rules that implement your theme variables. A `varsFile` option is required to identify relevant theme variables. The returned string is flat CSS with interpolation fields (ie: `<%= var %>`) wrapping theme variables.

This method requires the `node-sass` library installed as a peer dependency. Also note, variable names must pass through the Sass compiler as literals, therefore theme variables may NOT be used as function arguments (ie: `tint($this-will-explode, 10)`) or in math expressions (ie: `$sad-trombone * 0.5`).

```javascript
var sassThematic = require('sass-thematic');

// Async
sassThematic.renderThemeTemplate({
  file: './styles/main.scss',
  varsFile: './styles/_theme.scss',
  includePaths: ['./lib/'],
  templateOpen: '<%=',
  templateClose: '%>'
}, function(err, templateString) {
   console.log(templateString);
});

// Sync
var templateString = sassThematic.renderThemeTemplateSync({ ...options... });
```

## Full API Options

#### Required for all methods, one or both:

* **`file`**: String path to the main Sass file to load and parse. This may be an absolute path, or else a relative path from `cwd`.

* **`data`**: A Sass string to parse. You may still provide a `file` option as filepath context for mapping imports.

#### Required for theme parsing methods:

* **`varsFile`**: String path to a file containing all theme variables. This may be an absolute path, or else a relative path from `cwd`. This file must contain all theme variable definitions, and nothing else.

#### Required for theme rendering methods, one of:

* **`themeFile`**: String path to a file containing all theme variables to render CSS with. This may be an absolute path, or else a relative path from `cwd`.

* **`themeData`**: String data containing Sass variable definitions for all theme variables to render CSS with. Should be formatted as `$color1: red; $color2: black;`.

#### Optional options:

* **`includePaths`**: Array of base paths to search while performing file lookups. These should be absolute directory paths, or else relative to `cwd`. This operates just like the `node-sass` option of the same name.

* **`cwd`**: Path of the directory to resolve `file`, `varsFile`, `themeFile`,  and `includePaths` references from. Uses `process.cwd()` by default.

* **`templateOpen`**: The opening token for template interpolation fields. Uses ERB-style `<%=` by default.

* **`templateClose`**: The closing token for template interpolation fields. Uses ERB-style `%>` by default.

* **`templateSnakeCase`**: Boolean. Set as `true` to transform all template variable names to `snake_case` (lowercase with underscores).

* **`outputStyle`**: For rendering methods, this option is passed through to the Sass compiler to define output format. See [node-sass](https://www.npmjs.com/package/node-sass) docs for possible values.

## Pruning

SassThematic currently supports the following basic implementations:

* Removing unthemed rulesets and declarations.
* Removing unthemed mixins and their `@include` implementation.
* Removing unthemed `@extend` implementations.
* Removing unthemed loops (`@for`, `@each`), with basic local variable inflection.

This tool is a self-acknowledged 90% system that attempts to provide good automation for conventional use cases. Sass is a complex and nuanced language, therefore all of these pruning implementations undoubtedly have holes. For best results, review the [tests specs](https://github.com/gmac/sass-thematic/tree/master/test/style/reduce) to see what capabilities exist, and moderate complexity while implementing theme variables.

## Webpack Plugin

SassThematic provides a Webpack plugin to live-compile theme assets during development. The system uses a hybrid loader/plugin within the Webpack build pipe. Example Webpack config:

```
var sassThematic = require('sass-thematic');

var config = {
  module: {
    loaders: [
      { test: /\.scss$/, loader: "style!css!sass!sass-thematic") }
    ]
  },
  plugins: [
    sassThematic.plugin({
      varsFile: '../styles/shared/_theme.scss',
      includePaths: ['./styles/shared'],
      cwd: __dirname,
      output: [{
        includeFiles: [
          'components/site/base/index.scss',
          'components/site/auth/index.scss'
        ],
        template: {
          filename: 'theme2.css',
          writePath: '/Users/gregmacwilliam/Desktop/',
          templateOpen: '<%= @theme[:',
          templateClose: '] %>',
          templateSnakeCase: true,
          outputStyle: 'compressed',
          banner: '<%= @theme[:theme_prefix] %>',
          footer: '<%= @theme[:theme_postfix] %>'
        },
        css: {
          filename: 'theme.css',
        }
      }]
    })
  ]
}
```

1. Add `sass-thematic` as the right-most Sass loader. This should be added **in addition to* a standard Sass loader. The SassThematic loader simply loads changed file sources; you'll still need the standard Sass loader to compile assets.

1. Add a `sassThematic.plugin(...)` call into the `plugins` list. The plugin options detail the build of one or more theme stylesheet files. Plugin options are as follows:

 - `varsFile`: required. The file of theme variable definitions.
 - `themeFile`: optional. The file of theme variables to render into CSS output.
 - `includePaths`: optional. An array of locations to include in file lookups.
 - `cwd`: optional. A current working directory reference to resolve relative paths from.
 - `output`: required. An object (or array of objects) detailing theme files to generate.
   - `output.includeFiles`: required. An _ordered_ array of Sass files to include in the theme stylesheet.
   - `output.template`: parameters detailing the render of this output's template. This template file will be published to your standard Webpack output location. Include a `writePath` option to also write the file to a custom output location (useful for writing templates into an app directory).
   - `output.css`: parameters detailing the render of this output's CSS stylesheet. This css asset will be published to your standard Webpack output location. Include a `writePath` option to also write the file to a custom output location.

## Gulp Pipe

It's pretty simple to setup a Gulp pipe that hooks multiple Sass entry point files into SassThematic. Use the following as a basic template:

```javascript
var gulp = require('gulp');
var vinyl = require('vinyl');
var through2 = require('through2');
var sassThematic = require('sass-thematic');

// SassThematic Gulp pipe:
function sassTheme(opts) {
  var output = '';
  return through2.obj(function(file, enc, done) {
      opts.file = file.path;
      opts.data = file.contents.toString('utf-8');

      sassThematic.parseThemeSass(opts, function(err, result) {
        output += result;
        done();
      });
    },
    function(done) {
      this.push(new vinyl({
        path: 'theme.scss',
        contents: new Buffer(output)
      }));
      done();
    });
}

// Then use it...
gulp.task('theme', function() {
  return gulp.src('components/**/index.scss')
    .pipe(sassTheme({ ... opts ...}))
    .pipe(gulp.dest('/path/to/output/dir'));
});
```

## Credit

This toolkit would be impossible without the hard work of [@tonyganch](https://github.com/tonyganch) on the [gonzales-pe](https://github.com/tonyganch/gonzales-pe) lexer, which provides the framework for intelligently dismantling Sass. Serious kudos.

Brought to you by [Vox Media](http://voxmedia.com). <img src="http://fonts.voxmedia.com/emoji/voxmedia.png" alt="" width="15" height="15" style="width:15px;">
