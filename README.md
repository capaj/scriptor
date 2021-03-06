Scriptor
========

[![NPM Version][npm-image]][npm-url]
[![NPM Downloads][downloads-image]][downloads-url]

## Introduction

Scriptor is a small library for dynamically loading, reloading and running scripts without having to restart the process.

A Script is defined as a file which can be 'required' by Node, be it a `.js`, `.json`, or any other installed extension.

It does this by exploiting the `module` module's classes and functions that Node uses internally. The only downside to this is that it compiles the scripts synchronously. So to make the impact of this as little as possible, all compilation and evaluation of scripts is done lazily. That means even if you create a Script instance, it will not load or compile the script until it is actually needed.

Additionally, with the use of the Manager class, Scriptor provides a way of referencing and calling other scripts relative to itself, allowing you to chain together complex tasks easily.

Scriptor will also watch files for changes and reload the scripts (albeit lazily), when said file changes, allowing for scripts that run up-to-date code.

## Purpose

The original purpose for this system was to create a framework for writing simple scripts that generate web pages. That is still in development, but being able to separate out, reuse and update code without having to restart my server process was key.

# Documentation

All documentation for this project is in TypeScript syntax for typed parameters.

- [Script](#script)
    - [`new Script(filename? : string, parent? : Module)`](#new-scriptfilename--string-parent--module---script)
    - [`.load(filename : string, watch? : boolean)`](#loadfilename--string-watch--boolean---script)
    - [`.call(...args : any[])`](#callargs--any---any)
    - [`.apply(args : any[])`](#applyargs--any---any)
    - [`.require(path : string)`](#requirepath--string---any)
    - [`.unload()`](#unload---boolean)
    - [`.reload()`](#reload---boolean)
    - [`.watch()`](#watch---boolean)
    - [`.unwatch()`](#unwatch---boolean)
    - [`.close(permanent? : boolean)`](#closepermanent--boolean)
    - [`.id`](#id---string)
    - [`.children`](#children---module)
    - [`.parent`](#parent---module)
    - [`.loaded`](#loaded---boolean)
    - [`.watched`](#watched---boolean)
    - [`.filename`](#filename---string)
    - Special functions when used with a Manager:
        - [`.reference(filename : string, ...args : any[])`](#referencefilename--string-args--any---any)
        - [`.reference_apply(filename : string, args : any[])`](#reference_applyfilename--string-args--any---any)
        - [`.reference_once(filename : string, ...args : any[])`](#reference_oncefilename--string-args--any---referee)
        - [`.include(filename : string)`](#includefilename--string---script)

- [Script Environment](#script-environment)
    - [`module.define(id? : string, deps? : string[], factory : Function)`](#moduledefineid--string-deps--string-factory--function)
    - [`module.reference(filename : string, ...args : any[])`](#modulereferencefilename--string-args--any---any)
    - [`module.reference_apply(filename : string, args : any[])`](#modulereference_applyfilename--string-args--any---any)
    - [`module.reference_once(filename : string, ...args : any[])`](#modulereference_oncefilename--string-args--any---referee)
    - [`module.include(filename : string)`](#moduleincludefilename--string---script)

- [Referee](#referee)
    - [`.value()`](#value---any)
    - [`.ran`](#ran---boolean)

- [Manager](#manager)
    - [`new Manager(grandParent? : Module)`](#new-managergrandparent--module---manager)
    - [`.add(filename : string, watch? : boolean)`](#addfilename--string-watch--boolean---script)
    - [`.remove(filename : string, close? : boolean)`](#removefilename--string-close--boolean---boolean)
    - [`.call(filename : string, ...args : any[])`](#callfilename--string-args--any---any)
    - [`.apply(filename : string, args : any[])`](#applyfilename--string-args--any---any)
    - [`.once(filename : string, ...args: any[])`](#oncefilename--string-args-any---referee)
    - [`.once_apply(filename : string, args : any[])`](#once_applyfilename--string-args--any---referee)
    - [`.get(filename : string)`](#getfilename--string---script)
    - [`.clear(close? : boolean)`](#clearclose--boolean)

- [Rational and Behavior](#rational-and-behavior)

- [Changelog](#changelog)

##Script

A Script is the central part of Scriptor. It represents a single executable script, and manages modules, file changes and cleanup of the script. It also takes care of injecting the extra values into `module` when compiled.

#####`new Script(filename? : string, parent? : Module)` -> `Script`

Creates a new Script instance, optionally loading it with the filename specified, and with the specified parent context. `parent` defaults to the Scriptor module context.

Example:
```javascript
    var Scriptor = require('scriptor');

    var script = new Scriptor.Script('something.js');

    var result = script.call(some, arguments, here);
```

If `filename` is given, the constructor calls `.load` for you with that filename.

<hr>

#####`.load(filename : string, watch? : boolean)` -> `Script`

Since Scriptor emphasizes lazy evaluation, all `.load` really does is mark the script instance as not loaded and tells it to load and evaluate `filename` whenever it is needed. So this operation has no performance hit.

It does set `.id` to the basename of this filename given, but that can be overwritten manually anytime after. See the documentation of [`.id`](#id---string) for more on that.

If `watch` is true, a file watcher is spawned to watch the script for changes. For more information on how Scriptor handles this situation, refer to [`.watch`](#watch).

<hr>

#####`.call(...args : any[])` -> `any`

`.call` evaluates the script with any arguments passed to it. What this really does is, if module.exports of the targeted script is a function, calls that function with the arguments given.

It reflects the behavior of `Function.prototype.call` but without the `this` argument.

Though if module.exports is an object or any other type it will simply return that, as it cannot be evaluated any further.

NOTE for React.js users: React.js classes are functions, so a factory function is required to return them.

Example:
```javascript
/**** HelloMessage.js ****/
module.define(['react'], function(React) {
    var HelloMessage = React.createClass({displayName: "HelloMessage",
        render: function() {
            return React.createElement("div", null, "Hello ", this.props.name);
        }
    });

    //amdefine sets module.exports to equal the function below
    return function() {
        return HelloMessage;
    }
});

/**** main.js ****/
var React = require('react');

var hello = new Scriptor.Script('HelloMessage.js');

var html = React.renderToStaticMarkup(React.createComponent(hello.call(), {name: "John"}));
```

<hr>

#####`.apply(args : any[])` -> `any`

To reflect a `Function`, a Script instance has a `.apply` method that takes an array to be applied to the function, similar to `Function.prototype.apply`, but again without the `this` argument.

Used together, `.call` and `.apply` can generally satisfy any common way of evaluating a script.

<hr>

#####`.require(path : string)` -> `any`

This is an alias to the native `require` function used within the script.

<hr>

#####`.unload()` -> `boolean`

Marks the script as not loaded and returns true if it was loaded before `.unload` was called, false otherwise.

This does not delete the script, remove the file watcher or anything other than mark it as not loaded. It doesn't even delete the cached module.exports.

Any calls to `.call` or `.apply` after this will re-load and re-evaluate the script.

To permanently close a script and make it unusable so it can be garbage collected, refer to [`.close`](#closepermanent--boolean).

<hr>

#####`.reload()` -> `boolean`

**_WARNING_**: Usage of this function is not recommended. Use `.unload` if possible.

`.reload` forces re-loading and re-evaluation of the script, synchronously. It's the only function that is not lazy, hence why it is not recommended. However, it can have use in some rare conditions. It is still recommended to use `.unload` to achieve the same end lazily.

<hr>

#####`.watch()` -> `boolean`

Creates a file watcher to watch for any changes of the target script. If a change is detected, such as actual file changes or file renaming, this script is marked as not loaded and, in the case of renaming, the `.filename` attribute is updated.

On the next invocation of `.call` or `.apply`, the script is then loaded and evaluated as usual, but with the changes accounted for.

Returns true if a file watcher was added, false if one already existed.

If the script is updated often but invoked infrequently, the lazy evaluation of the script means there is almost no performance hit caused by repeated file changes.

In the event of file deletion, the script is unloaded, unwatched and `.filename` is set to null. Due to the limitations of the file watcher system, it cannot automatically pick up on a new file with the same name if it were to be placed back. Probably because the new file would have a different inode value or something.

After file deletions, a call to `.load` will set things back up correctly.

NOTE: For simple renames without changing file content, the script is not unloaded, but the `.filename` attribute is updated accordingly.

<hr>

#####`.unwatch()` -> `boolean`

Removes a file watcher if one existed.

Returns true if it was able to remove an existing file watcher, false if one didn't exist in the first place.

NOTE: After a call to `.unwatch`, file watching is disabled, and any file changes will not be reflected.

<hr>

#####`.close(permanent? : boolean)`

Unloads and unwatches the script.

`permanent` defaults to true.

If `permanent` is true, the script module is deleted from the parent module, and from the Script instance. After a call to `.close(true)`, the script instance is rendered useless. This should only be used when the script is not needed again. In order to invoke the script again, a new instance must be created.

<hr>

#####`.id` <-> `string`

This attribute is a simple identifier used by the Node.js module system for stuff. Calls to `.load` set it to the basename of the filename given, but it can be overwritten manually.

Example:
```javascript

var script = new Scriptor.Script('something.js');

script.id = "My Script";

script.call(); //'module.id' within the script context is "My Script"
```

<hr>

#####`.children` -> `Module[]`

Although this is rare, a script is still technically a module underneath, so it is possible for it to have children. This is allows access to them.

<hr>

#####`.parent` -> `Module`

Reference to the parent module.

<hr>

#####`.loaded` -> `boolean`

True if the script is loaded and compiled, or false if it is not.

<hr>

#####`.watched` -> `boolean`

True if the script file is being watched, false otherwise.

<hr>

#####`.filename` -> `string`

Returns the filename of the script. If no file has been given or the file has been deleted, this equals null.

<hr>

###Special functions when used with a Manager:

These functions are added to the Script when the scripts are created via a Manager instance.

Unlike the manager functions, these work with filenames relative to the script, which make it easier to work with complex directory structures where dealing with a root directory is way too much of a hassle. Of course absolute paths work as well.

#####`.reference(filename : string, ...args : any[])` -> `any`

This is effectively a call to `<manager>.add(filename).call(...args)`

It is useful when referring to other scripts within a Manager.

<hr>

#####`.reference_apply(filename : string, args : any[])` -> `any`

This is effectively a call to `<manager>.add(filename).apply(args)`

It is useful when referring to other scripts within a Manager.

<hr>

#####`.reference_once(filename : string, ...args : any[])` -> `Referee`

This can be a very powerful system function if used correctly. It basically evaluates a script ONCE, and returns a reference to the result in the form of a `Referee` class instance. By calling `.value()` on the Referee the returned value of the script can be accessed.

What is powerful about this is that even though it acts as a static variable, it takes into account file changes like the rest of Scriptor.

When a script referenced by `.reference_once` changes, the Referee picks up that change and re-evaluates it with the arguments that were initially given to it by `.reference_once`.

It also does this lazily, too. If the Referee has not invoked the script, it will the next time `.value()` is called.

A particularly good use for this is configuration files. As stated before, Scriptor can load anything `require` can, including JSON files.

Example:
```javascript
/**** Inside a script ****/
var config = module.reference_once('./configs/main.json');

var _ = require('lodash');

module.exports = function() {
    return _.extend(config.value(), {
        myCustomConfig: 'Hello'
    });
};
```

The above example, as used within a script, allows for a modified configuration to be returned, but with the config variable being static throughout the duration of the program unless `./configs/main.json` is changed, at which point it is marked stale and re-evaluated and stored as static until any further changes.

Although it can be a bit difficult to use concisely, automatically updating static variables are quite useful.

Furthermore, any more calls to `.reference_once` from anywhere else for the file `./configs/main.json` will return the exact same Referee instance, and will have access to the same automatically updating static value.

<hr>

#####`.include(filename : string)` -> `Script`

This is effectively a call to `<manager>.add(filename)`

It is useful when referring to other scripts within the same manager, and when you need access to the script itself instead of just invoking it.

<hr>

##Script Environment

Although the scripts are run by the native Node.js module system, Scriptor adds a couple functions into it to make things easier.

#####`module.define(id? : string, deps? : string[], factory : Function)`

This is arguably not needed, but is useful nonetheless. Using the amdefine library, Scriptor injects this function into the module before running the script, allowing the script to use this.

It's not quite as elegant as just having `define` by itself, but it had to work within the bounds of the module system.

<hr>

#####`module.reference(filename : string, ...args : any[])` -> `any`

This is alias for `<script>.reference`, with script being the Script instance that runs the code.

<hr>

#####`module.reference_apply(filename : string, args : any[])` -> `any`

This is alias for `<script>.reference_apply`, with script being the Script instance that runs the code.

<hr>

#####`module.reference_once(filename : string, ...args : any[])` -> `Referee`

This is alias for `<script>.reference_once`, with script being the Script instance that runs the code.

<hr>

#####`module.include(filename : string)` -> `Script`

This is alias for `<script>.include`, with script being the Script instance that runs the code.

<hr>

##Referee

The Referee class is a tiny wrapper around the behavior for `reference_once`. It keeps track of the initial arguments, the statically cached value and updating it when the script changes.

#####`.value()` -> `any`

Effectively 'dereferences' the reference and returns the value the script returned.

Due to the lazy evaluation nature of Scriptor, `.value()` might load and evaluate the script if need be. Though usually it doesn't need to and returns the cached value immediately.

<hr>

#####`.ran` -> `boolean`

True if the script has already run (and therefore the value is already cached), false if the script has not run.

<hr>

##Manager

Scriptor provides a Manager class that can effectively coordinate many inter-referencing scripts together with little effort.

#####`new Manager(grandParent? : Module)` -> `Manager`

Create a new Manager instance, optionally giving it a Module it will use as a grandparent when creating child scripts.

Internally it creates a single Module representing itself that it gives to child scrips as their parent. This is so that many scripts being coordinated will not pollute the real module, but instead a single-purpose intermediary module.

<hr>

#####`.add(filename : string, watch? : boolean)` -> `Script`

This will create a new Script instance with `filename` as the script file,
and set up file watching on it.

`watch` is true by default.

If a script already exists in the Manager, a new one is not created and the old one is returned. Additionally, `watch` is an additive argument. Setting it to false on an already added and watched script will not unwatch it.

`.add` can be used as a shortcut for adding and/or getting scripts in the same function.

<hr>

#####`.remove(filename : string, close? : boolean)` -> `boolean`

Removes a script from the Manager instance.

If `close` is true, it will permanently close the script pointed to by `filename` according to the specification give by [`Script.close`](#closepermanent--boolean)

`close` defaults to true.

<hr>

#####`.call(filename : string, ...args : any[])` -> `any`

Equivalent to `.add(filename).call(...args)`

<hr>

#####`.apply(filename : string, args : any[])` -> `any`

Equivalent to `.add(filename).apply(args)`

<hr>

#####`.once(filename : string, ...args: any[])` -> `Referee`

Although this has the same effect as `reference_once` from a Script instance, it's adapted for use in the Manager. It behaves exactly the same otherwise.

<hr>

#####`.once_apply(filename : string, args : any[])` -> `Referee`

Same as the above, but allows an array of arguments instead of using variable argument function magic.

<hr>

#####`.get(filename : string)` -> `Script`

Will return the Script instance stored in the Manager. If one does not exist, undefined is returned.

#####`.clear(close? : boolean)`

This will clear out the Manager and reset it.

If `close` is true, the scripts within are closed according to the specifications of [`Script.close()'](#closepermanent--boolean).

`close` defaults to true.

<hr>

##Rational and Behavior

This project is probably the most interesting single library I've written. It took four or five rewrites to get it to be as concise and useful as it is now.

One of the most important implementation details of this is the exploitation of the Node.js module module (not a typo). Internally, `require` and other things use the module module to load and compile files to be returned. It compiles and evaluates them in a semi-contained context that was perfect for server-side scripting systems. However, require caches everything, so a call to require the second time doesn't reload the script, otherwise Node would be slow and bloated quite a bit.

But, what really sold me on trying to use it was the module system itself. Not just that it runs it, but that it gives the 'module' variable to the script and I can create my own and give them custom attributes (like `module.reference`). All that combined, it was the perfect match.

So, I reverse-engineered almost all of the node module system down to its core, created TypeScript type definitions of all its internals ([See Here](https://github.com/novacrazy/node-module)) and figured out just how easy it was.

Plus, with the module API locked down with very few changes in the last couple years, I was able to create Scriptor.

However, the only bad part was that the module module, because it is almost always used at the startup of Node in `require` calls, is synchronous. So, a major design emphasis of Scriptor was that everything be lazily evaluated. That way even if a file changes thousands of times, it is only even loaded, compiled and evaluated when it is actually needed. Thereby reducing unexpected delays and generally making the system more responsive.

As I stated in the introduction, I am actively using this project within my own server system, and so far it is incredibly useful. Additionally, because everything, once compiled, is effectively just functions, calling a top level script is the same as one large and nested function in terms of performance, depending on what your scripts actually have in them. It's certainly better than anything else.

I lost a big chunk of latter part of this explanation when my IDE crashed parsing out this large markdown file when I accidentally clicked 'preview'. I may update this section later if I have more to add.

<hr>

##Changelog

#####1.0.1
* Fixed a few typos in README.md
* Added npm badges

#####1.0.0

* First full release
* API improvements
* Full documentation
* A couple bugfixes
    * Referee wasn't actually marking the result as ran (fixed)

#####pre-1.0.0
* development

[npm-image]: https://img.shields.io/npm/v/scriptor.svg?style=flat
[npm-url]: https://npmjs.org/package/scriptor
[downloads-image]: https://img.shields.io/npm/dm/scriptor.svg?style=flat
[downloads-url]: https://npmjs.org/package/scriptor
