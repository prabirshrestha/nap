# Node Asset Packager

**nap is currently in an early alpha development stage. Please feel free to take a look around the source code to get an idea of where the project is going.**

Compiling, packaging, minifying, and compressing your client side assets got you all strung out? Relax, take a nap.

(nap) Node Asset Packager is a module inspired by [Jammit](http://documentcloud.github.com/jammit/) and [Connect Asset Manager](https://github.com/mape/connect-assetmanager) that helps compile and package your assets including stylesheets, javascripts, and javascript templates.

## Example

````coffeescript
nap = require 'nap'

assets = 
  
  js:

    preManipulate:
      '*': [nap.compileCoffeescript]
    postManipulate: 
      'production': [nap.uglifyJS]

    backbone: [
      'app/coffeescripts/models/**/*.coffee'
      'app/coffeescripts/views/**/*.coffee'
      'app/coffeescripts/routers/**/*.coffee'
    ]

  css:

    preManipulate:
      '*': [nap.compileStylus]
    postManipulate: 
      'production': [nap.yuiCssMin]

    all: ['public/stylesheets/blueprint.css', 'app/stylesheets/**/*.styl']

  jst:
  
    preManipulate:
      '*': [nap.packageJST]
    postManipulate: 
      '*': [nap.prependJST("require('jade').compile")]

    templates: [
      'app/templates/index.jade'
      'app/templates/footer.jade'
    ]

switch process.env.NODE_ENV
  when 'production' then nap.package assets, 'public/assets'
  when 'development'then nap.watch assets, 'public/assets'
````

Then simply reference your packages in your layout

````html
<script type='text/javascript' src='assets/backbone.js'>
<link rel="stylesheet" type="text/css" href="assets/all.css" />
<script type='text/javascript' src='assets/templates.jst'>
````

This example...

* Compiles coffeescripts in various folders, merges & minifies the outputted js with uglifyJS
* Compiles stylus in `app/stylesheets` dir, merges blueprint.css with the compiled stylus, and finally minifies the result with YUI.
* Packages two jade templates similar to how Jammit does, using the jade client side compiler function when called (See _manipulators_ below).
* Outputs backbone.js, all.css, and template.jst package files to `public/assets` once in _production_
* Watches for any file changes in any of the packages, and recompiles that package in _development_ 


## Installation

    $ npm install nap
    
## Explaining nap

nap provides a library to manipulate, package, and finally output your client side assets in to a directory of your project. Give nap a well formatted asset object describing the packages and nap will output them into a directory of your choice.

There are currently two functions that take a nap asset object.

````coffeescript
# Runs the magic once and outputs the given assets to public/assets
nap.package assets, 'public/assets'

# Watches for file changes on a specific file and only re-compiles that package
nap.watch assets, 'output/path/of/choice'
````

## Examples of nap asset objects

_Package & minify vendor javascripts_

````coffeescript
assets =

  js:
    postManipulate: 
      '*': [nap.uglifyJS]

    vendor: ['public/javascripts/vendor/**/*.js']    
````
    
_Compile & package a bunch of coffeescripts. Minify only in production_

````coffeescript
assets =

  js:
    preManipulate:
      '*': [nap.compileCoffeescript]
    postManipulate: 
      'production': [nap.uglifyJS]

    mvc: [
      'app/coffeescripts/models/**/*.coffee'
      'app/coffeescripts/views/**/*.coffee'
      'app/coffeescripts/controllers/**/*.coffee'
    ]
````      

_Compile stylus into a CSS package_

````coffeescript
assets =

  css:
    preManipulate:
      '*': [nap.compileStylus]
    postManipulate: 
      'production': [nap.yuiCssMin]

    plugins: [
      'app/stylus/plugins/**/*.styl'
      'app/stylus/plugins/**/*.styl'
    ]
````

_Compile javascript templates using Jade's client side compiler function_

````coffeescript
assets =

  jst:
    preManipulate:
      '*': [nap.packageJST]
    postManipulate: 
      '*': [nap.prependJST("require('jade').compile")]

    plugins: [
      'app/templates/index.jade'
      'app/templates/footer.jade'
    ]
````

## Explaining the asset object

The assets object consists of a couple layers. First level is the package extensions. (_.js_, _.css_, and _.jst_) These simply determine the package file extensions and groups packages and manipulators together. Inside an extension you can specify keys preManipulate, postManipulate, and from then on each key describes a package.

## Packages

Packages are any key inside an extension not labeled preManipulate or postManipulate. (Don't name a package preManipulate.css) Packages are simply an array of file path strings. You may also pass wild cards in the form of `**/*` to recursively add all files in that directory or `/*` for just all files one level deep in that directory.
  
````coffeescript
# Recursively add all files in the folder app/javascripts/vendor/ to be ouput to vendor.js
assets =
  
  js:
    
    vendor: ['app/javascripts/vendor/**/*.js']
````

## Manipulators

preManipulate and postManipulate are an array of functions that are run, in order, on the files to be packaged. preManipulate is passed `(contents, filename)` and is run on each individual file in a package before being merged. postManipulate is passed `(contents)` and is run after each file in a package is merged.  
    
````coffeescript
# Compiles any coffeescript files with the .coffee extension, prepends a comment denoting it being a compiled file, 
# concatenates all of the files into one, then runs uglifyJS on the merged files.
assets =

  js:
    preManipulate:
      '*': [
        ((contents, filename) ->
          if filename? and filename.match(/.coffee$/)?
            return require('coffee-script').compile contents
          else
            return contents
        ),
        ((contents, filename)) ->
          if filename? and filename.match(/.coffee$/)?
            return "// This file was a compiled coffeescript file.\n" + contents
          else
            return contents
        )
      ]
    
    postManipulate: 
      '*': [nap.uglifyJS]

    mvc: [
      'app/coffeescripts/models/**/*.coffee'
      'app/coffeescripts/views/**/*.coffee'
      'app/coffeescripts/controllers/**/*.coffee'
    ]
````

## nap Manipulators

Although you can pass any custom manipulator function, nap provides a handful of common manipulator functions. Simply require nap `nap = require 'nap'` and pass any of the following namespaced functions.

### Pre-manipulators

Run the coffeescript compiler on files with the .coffee extension
    
    nap.compileCoffeescript
    
Run the stylus compiler on any of files with the .styl extension
    
    nap.compileStylus
    
Package the contents of the file into `window.JST['file/path']` namespaces. The path is determined by the folder path where the file resides, beginning from a templates folder if provided, with the extension removed. 

e.g. A file in `app/templates/home/index.jade` will be packaged as `window.JST['home/index'] = JSTCompile('h1 Hello World');`
    
    nap.packageJST
    
### Post-manipulators

Run uglifyJS on the merged files
    
    nap.uglifyJS
    
Run the YUI css compressor on the merged files
    
    nap.yuiCssMin
    
Used in conjunction with nap.packageJST to determine what client-side javascript template compiler function you want to use. Pass the function name as a string. e.g. `nap.prependJST('Haml')` or `nap.prependJST('Mustache.to_html')`
    
    nap.prependJST('Haml')
      
## To run tests

nap uses [Jasmine-node](https://github.com/mhevery/jasmine-node) for testing. Simply run the jasmine-node command with the coffeescript flag

    jasmine-node spec --coffee
    

## License

(The MIT License)

Copyright (c) 2009-2010 Craig Spaeth <craigspaeth@gmail.com>

Permission is hereby granted, free of charge, to any person obtaining a copy of this software and associated documentation files (the 'Software'), to deal in the Software without restriction, including without limitation the rights to use, copy, modify, merge, publish, distribute, sublicense, and/or sell copies of the Software, and to permit persons to whom the Software is furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED 'AS IS', WITHOUT WARRANTY OF ANY KIND, EXPRESS OR IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY, FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM, OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE SOFTWARE.