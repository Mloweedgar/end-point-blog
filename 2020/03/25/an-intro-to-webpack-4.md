---
author: "Kevin Campusano"
title: "An Introduction to Webpack 4: Setting up a modern, modular JavaScript front end application"
tags: web development, front end development, javascript, webpack, module bundler, babel, ES2015, ES6
---

![Banner](an-intro-to-webpack-4/banner.png)

Image taken from https://webpack.js.org/

I've got a confession to make: Even though I've developed many JavaScript-heavy, client side projects with complex build pipelines, I've always been somewhat confused by the engine that drives it all under the hood: [Webpack](https://webpack.js.org).

Up until now, when it came to setting up a build system for front end development, I always deferred to some framework's default set up or some recipes discovered after some Googling or StackOverflow-ing. I never really understood Webpack in a detail level where I could feel comfortable reading, understanding and modifying a config file.

This "learn enough to be effective" approach has served me well so far and it works great for being able to get something working, while also spending time efficiently. When everything works as it should, that is. This approach starts to fall apart when weird, more obscure issues pop up and you don't know enough about the underlying system concepts to get a good idea of what could've gone wrong. Which can sometimes lead to frustrating Googling sessions accompanied with a healthy dose of trial and error. Ask me how I know...

Well, all that ends today. I've decided to go back to basics with Webpack and learn about the underlying concepts, components and basic configuration. Spoiler alert: it's all super simple stuff.

Let's dive in.

### The problem that Webpack solves

Webpack is a module bundler. That means that its main purpose is to take a bunch of disparate files and "bundling" them together into single, aggregated files. Why would we want to do this? Well, for modularity. I.e. to be able to write code that's modular.

Writing modular code is not as easy in JavaScript that runs in a browser as it is in other languages or environments. Traditionally, the way to achieve good modularity in the web front end has been via including separate scripts via multiple `<script>` tags within HTML files. This approach comes with its own host of problems. Things like the order in which the scripts are included suddenly matter, because the browser executes them top to bottom, which means that you have to be very careful to include them in an order where dependencies of the later files are included first. Also, this approach encourages the pollution of the global scope, where every script declares some global variables which are then used by other scripts. This is problematic because it is not clear which scripts depend on which ones. In other words, dependencies become implicit and hard to track. Unit testing becomes harder as well, since you need to mock these dependencies at a global scope. You also run the risk of some scripts overriding these global resources anytime. It's just not very clean.

Over the years, solutions have been devised by the community to tackle this issue, which have taken the form of module loaders like [CommonJS](http://www.commonjs.org/) and [AMD](https://github.com/amdjs/amdjs-api/blob/master/AMD.md). Finally, [ES2015](https://babeljs.io/docs/en/learn/) introduced a native module loading system into the language itself via the `import` and `export` statements. The life of JavaScript in the browser is not so easy though, as these new specifications take time to implement by the various browser vendors. This is where Webpack comes in. Part of its offerings is the ability to use this standard, modern module loading mechanism without having to worry about browser compatibility. Which unlocks the potential for front end developers to write modern, beautiful, modularized JavaScript. This is huge.

### Concepts: Entry Points and Output

Now, what does that look like in Webpack terms? Well, we define an Entry Point, an Output specification, and let Webpack do its thing.

This is a good time to formally introduce two of Webpack's core concepts. First, we have Entry Points.

Entry Points represent the starting point of execution for your app or page, and as such, serve as the starting file for Webpack to begin building up your bundle. These are normally JavaScript files that bootstrap the execution of whatever application or page that you are developing. From a software design perspective, they tend to import many other modules and start up the scripts and instantiate the objects that actually run the application logic.

So, for example, if you had a file named `index.js`, which has some front end logic and uses some utility classes living in separate files like `SomeServiceClass.js` or `ApiClient.js`; then `index.js` is a great candidate for being an Entry Point for your application or page. Because it is the one singular file that calls upon all of the other dependencies/modules.

We also have Output. The Output is the result of a Webpack bundling operation. When Webpack takes our Entry Points, it builds and compiles their corresponding dependencies and produces a single JavaScript file that can be directly included into our page or app via a `<script>` tag. This file is our Output. This is the only file that needs to be included, because Webpack took all the separate dependencies and bundled them together in one single package.

### Introducing our demo app and its problems

But let me show rather than tell. Consider a simple calculator application, whose source code file structure looks like this:

```
.
├── index.html
└── js
    ├── calculator
    │   ├── calculator.js
    │   └── operations
    │       ├── addition.js
    │       ├── division.js
    │       ├── multiplication.js
    │       ├── operation.js
    │       └── subtraction.js
    └── index.js
```

You can explore the source code for this small application here: https://github.com/megakevin/end-point-blog-webpack-intro/tree/master/original. Feel free to go ahead and download it if you want to work along with me.

I've called the app the "Automatic Calculator" (not to be confused with its much less powerful alternative, the "manual" calculator!) and you will be able to figure out its general architecture pretty quickly.

In the root directory, we've got `index.html` which contains the GUI for our app. Then, all the behavior is inside the `js` directory.

`index.html` is pretty straightforward. It's got a simple form with two fields for typing in numbers, and a button that will "automatically" run a few arithmetic operations on those numbers. The results are presented right there in the page just a few pixels below the form.

For the purposes of this post, the interesting part of that file comes near the bottom, where we include all of our JavaScript logic. It looks like this:

```html
<script src="js/calculator/operations/operation.js"></script>
<script src="js/calculator/operations/addition.js"></script>
<script src="js/calculator/operations/subtraction.js"></script>
<script src="js/calculator/operations/multiplication.js"></script>
<script src="js/calculator/operations/division.js"></script>
<script src="js/calculator/calculator.js"></script>
<script src="js/index.js"></script>
```

As you can see, our little Automatic Calculator logic is separated into a series of files. And right now we start seeing some of the drawbacks of not using any sort of module loading for our app. This page has to include all of the script files that it needs separately. What's worse, the order in which they are included matters. For example, since the `js/calculator/calculator.js` file depends on the `js/calculator/operations/multiplication.js` file to work, `multiplication.js` needs to be included first. If not, the page would break. From this page's perspective, it would be much easier and cleaner if it could just include one file, one "bundle" that has everything it needs.

If we look at our script files, we see more related problems. Consider `js/index.js`, for example. This is the file that starts up the app. It defines an App class which it then instantiates and runs. Here's what it looks like (explore the source code in the git repo if you want to see the whole thing):

```js
class App {
    constructor() {
        this.calculator = new Calculator();

        /* Omitted */
    }

    run() {
        /* Omitted */
    }
}

console.log("Starting application");

let app = new App();
app.run();

console.log("Application started");
```

The `App` class' constructor is creating a new instance of the `Calculator` class. The problem is that, from the point of view of the reader of this file, it's doing so out of thin air. There's no indication whatsoever that this file depends on and uses the `Calculator` class. It is available here only because the file that contains that class happens to be included in a `<script>` tag in the same page that is using `js/index.js`. That's hard to read and maintain as the dependency is implicit in a place where it should be explicit. Imagine if `js/index.js` was a couple hundred lines bigger and had a few dozen more dependencies. That'd be very hard to manage. One would have to read the entire file to get an idea of how the code is structured. To be able to reason about the code from a higher level, one would have to go to the really low level. Too much cognitive overhead.

The same thing happens in the `js/calculator/calculator.js` file. It defines the `Calculator` class and that class depends on other classes (`Addition`, `Subtraction`, etc). Which are also defined in other JavaScript files but the `js/calculator/calculator.js` does not explicitly say that those are dependencies that it needs. They are called up out of thin air. Everyone is trusting on `index.html` to include all the separate script files where these classes are defined and that it does so in the proper order. Too much responsibility for little old `index.html`. And also, what would happen if somebody wanted to reuse the `Calculator` class in another page for instance? That developer would have to know all of the dependencies for that class and include them manually in the new page. What if the file were much bigger, with more dependencies? That can get really ugly really quickly.

Luckily for us, those are exactly the kinds of problems that Webpack helps us deal with. So let's start refactoring this little app so that it can take advantage of a Webpack based build process.

### Installing Webpack into our project

> If you are following along and downloaded the source code from the GitHub repo; then whenever I say "root", I mean that repo's `original` directory. That's where the original version of the Automatic Calculator app's source code lives. As it is before introducing Webpack. You can work directly from there or take the contents of that directory and put them wherever it is most comfortable to you. The `final` directory contains the source code as it will be by the end of this post.

The first thing we need to do is install Webpack into our project. That's super easy if you already have [NodeJS](https://nodejs.org/) and [NPM](https://www.npmjs.com/) installed. How to install NodeJS and NPM is out of the scope of this discussion, so I would recommend following NodeJS's documentation to get them installed.

Once you have that, go to our project's root and run `npm init -y`. This will create a `package.json` file with some default configuration. This effectively makes our code a proper NodeJS project.

After that, installing Webpack is as easy as going to our project's root and running:

```
npm install webpack webpack-cli --save-dev
```

That will create a new `node_modules` directory and install both the `webpack` and `webpack-cli` packages as development dependencies. They are installed as dev dependencies because, in production, our app won't actually need Webpack to run properly. Webpack will only help us build the deployment assets for our app, and that happens at development time.

### The Webpack configuration file

Now, we need to create a `webpack.config.js` file which tells Webpack how to take our files and produce said deployment assets. I.e. the "compiled" bundle, the Output. Here's what a simple config file tailored to our app would look like.

```js
const path = require('path');

module.exports = {
    mode: "development",
    entry: "./src/index.js",
    output: {
        filename: "index.js",
        path: path.resolve(__dirname, "dist"),
    },
};
```

Let's discuss it line by line.

First, there's `const path = require('path');` which is just including the NodeJS native `path` package that the config file uses further down below.

Then there's the `module.exports` definition. Conveniently, Webpack's configuration values are organized within a JavaScript object. `module.exports` is what contains that object.

Now we have the actual Webpack build settings. The `mode` field can be either `development`, `production` or `none`. Not really super important for us right now, but depending on what you set here, Webpack will apply certain optimizations to the bundles.

The `entry` field defines the Entry Point for the build process. Like discussed before, this is the first file that Webpack will start from when figuring out the whole [dependency graph](https://webpack.js.org/concepts/dependency-graph/). That is, all of the files that, via `import` and `export` statements (more on that later), specify that they need one another to work. In our app, our dependency graph looks like this:

![Image](an-intro-to-webpack-4/dependency-graph.png)

That is, `index.js` depends on `calculator.js`; `calculator.js` in turn depends on `addition.js`, `subtraction.js`, `multiplication.js` and `division.js`; and all of the later four ones depend on `operation.js`. Here, we have specified `./src/index.js` as our Entry Point. But wait, our `index.js` file lives inside our `js` dir. What gives? We'll change that soon. When we prepare our files to use ES2015 modules and have an overall more conventional organization, as far as Webpack is concerned.

Finally, with the `output` field, we tell Webpack what we want it to produce after bundling together all those files. In this case, we've configured it to produce a file called `index.js` inside a `dist` directory.

### Refactoring our project to use ES2015 modules

Like discussed before, Webpack allows us to express our dependencies using the now standard and natively supported on the language level, ES2015 `import` and `export` statements. Let's do that. This is super easy, and you will understand it immediately if you have used any other language that supports modules like C#, Java, Python, PHP... Yeah, pretty much any other language out there BUT JavaScript.

Anyway, we have to add this line at the beginning of `index.js`:

```js
import Calculator from "./calculator/calculator.js";
```

Nice, this is an explicit dependency declaration for our `index.js` file. Now Webpack (and readers on the code!) can know that this file uses the Calculator class. And like that, we go file by file adding dependencies. In `calculator.js`, we do:

```js
import Addition from "./operations/addition.js";
import Subtraction from "./operations/subtraction.js";
import Multiplication from "./operations/multiplication.js";
import Division from "./operations/division.js";
```

In all of those four files, we do:

```js
import Operation from "./operation.js";
```

It's all pretty self-explanatory. The syntax is `import <CLASS_NAME> from "<RELATIVE_FILE_PATH>";` where the path is relative to the location of the file where we're adding the `import` statement.

Now, `import` statements are for specifying which other files a given file needs to work. To allow the code elements that are defined within a file to be imported by others though, they need to be "exported". We do that by annotating the code elements that we want to make available with the `export` statement. `export` basically specifies what parts of a module are available for others to use.

Our code is factored in a very simple way, where the only thing defined in a given file is a class. So, in order for all the `import` statements that we added to work, we just need to go file by file doing changes like this:

```diff
-class Calculator {
+export default class Calculator {
```

All we did was add the `export` and `default` keywords to the definition of the `Calculator` class in `calculator.js`. `export` makes it so we are allowed to import that class elsewhere, and `default` allows us to use the style of import that we used before: the `import Calculator from "./calculator/calculator.js";` one.

> By the way, yes, there are other styles of `import` and the `default` keyword is optional. To learn more about `import` and `export ` statements and JavaScript modules in general, I'd recommend a few resources from MDN: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide/Modules, https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/import, https://developer.mozilla.org/en-US/docs/web/javascript/reference/statements/export.

Alright so after adding `export default` to all of our class definitions, except for the `App` class defined in `index.js` (we don't need to export it because we don't import it anywhere), we almost have all the pieces of the puzzle ready for our Webpack build to work.

The only thing left is to rename our `js` directory to `src` to match our Webpack configuration. No particular reason for using `src` instead of `js` other than it being a good way to clearly separate the location of the source code versus the location of the compiled assets. Which will live in `dist`.

Anyway, with that rename done, you can go ahead and run

```
npx webpack --config webpack.config.js
```

As a result of that command, you should see a new `dist` directory with a single `index.js` file in it. Just like we specified as the output in our Webpack configuration file.

And that's it! We have set up a Webpack build process for our app. Don't forget to change the `index.html` page so that it only includes this new compiled asset. With something like this, at the end of the `<body>`:

```html
<script src="dist/index.js"></script>
```

You should be able to open up the page in your browser and marvel at what we've accomplished:

![Image](an-intro-to-webpack-4/the-app.png)

### The other problem that Webpack solves

Ok, we've made a great accomplishment here. We've managed to leverage Webpack to write modular JavaScript. The importance of this cannot be understated. However, Webpack offers us much more. Another problem that has historically plagued front end web development is compatibility across multiple browsers. Webpack helps us with that as well via Loaders.

### Concepts: Loaders

In a nutshell, Loaders are components that Webpack uses to transform source code files during its bundling process, in order to make them available to the app. They basically allow Webpack to turn “things”, other than JavaScript, into modules that can be used by the application via `import`. By itself, Webpack only knows how to process JavaScript and JSON files. Using Loaders however, Webpack can also process other types of files like CSS and its various flavors like LESS or SASS; or even JavaScript derived languages like TypeScript or CoffeeScript.

### Writing modern JavaScript with Babel

Here's where the browser compatibility solution that I alluded to earlier comes into play: the `babel-loader`. This loader makes it so we can write bleeding edge JavaScript, with all the newest features of the language, without having to worry if our target browsers support them.

[Babel](https://babeljs.io/) is a compiler that takes code written with the latest version of the language and turns it into plain old JavaScript that can run anywhere. `babel-loader` is how Babel integrates with Webpack. I like being able to take advantage of JavaScript's latest features, so, to me, setting up Babel in any new project is a must. Luckily for us, with Webpack, setting it up is easy.

> You can learn more about Babel and the latest JavaScript features in https://babeljs.io/docs/en/ and https://babeljs.io/docs/en/learn.

Like everything else, Babel is distributed as an NPM package. So let's go ahead and install it with

```
npm install --save-dev babel-loader @babel/core @babel/preset-env
```

This will install the core Babel package as well as the Loader for Webpack, and the `env` preset.

> The concept of "preset" is actually a Babel related one, not really having anything to do with Webpack. You can learn more about them here https://babeljs.io/docs/en/babel-preset-env, but, for our purposes here, suffice it to say that Babel presets are a specification of the features that are available to use. There are many presets (i.e. feature sets) against which you can configure Babel to compile. `env` is just a very convenient one that provides support for all the latest language features.

Again, we're installing these as dev only dependencies because they are not needed for the app at runtime. Only at build time.

Now, we go to our `webpack.config.js` file and add these lines:

```js
module.exports = {
    // ...
    module: {
        rules: [
            { test: /\.js$/, exclude: /node_modules/, loader: "babel-loader" },
        ]
    }
};
```

This is how we specify that we want the `babel-loader` to transform our JavaScript files before Webpack bundles them together. Inside `module.rules`, there's an array of objects. Each of the elements of that array is a rule specifying, on one hand, which files it applies to, via the regular expression in the `test` field; and, on the other hand, which loader will be used to process them, via the `loader` field. The `exclude` field makes sure that files under our `node_modules` directory are not affected by `babel-loader`. We don't want to be transforming the packages we downloaded from NPM after all, those are ready to use as they are. We only want to transform our own code.

In summary, this rule tells Webpack to “run all `.js` files by `babel-loader` except for those inside `node_modules`”. `babel-loader` will make sure to transform them into plain old JavaScript before giving them back to Webpack for bundling.

Finally, Babel itself requires a little bit of configuration. So let's give it what it needs by creating a `.babelrc` file in our project's root with these contents:

```js
{
    "presets": ["@babel/preset-env"]
}
```

Pretty self-explanatory. This configuration tells Babel to use the `env` preset when processing our files.

Now run `npx webpack --config webpack.config.js` and hope for the best. Just kidding! Everything should work like a charm and with that, we have just unlocked JavaScript's full potential for our project. Open the page again and you will see nothing has changed. What we have gained is the ability to write modern JavaScript without having to worry about compatibility.

### Bonus: Multiple Entry Points

When building a [SPA](https://en.wikipedia.org/wiki/Single-page_application), a single Entry Point is appropriate. However, a lot of times we have apps with several pages, each one of which is a sort of advanced front end application in its own right. From a Webpack perspective, apps like that have multiple Entry Points. One for each page. Because each page has its own set of client side code that runs independently from each other. They may share common parts under the hood (via class or library reuse, for example) but the main, initial script is different.

Let's consider our calculator. So far, that app has only one page. Imagine we want to add a new one. Let's say, an admin control panel. To make that happen, let's add new `admin.html` and `src/admin.js` files to the project. Their contents don't matter. All I need is for them to exist in order to illustrate the multiple Entry Points capability.

As far as Webpack is concerned, we can configure the build process to support that style of application by updating our `webpack.config.js` like so:

```diff
 module.exports = {
     mode: "development",
-    entry: "./src/index.js",
+    entry : {
+        index: "./src/index.js",
+        admin: "./src/admin.js"
+    },
     output: {
-        filename: "index.js",
+        filename: "[name].js",
         path: path.resolve(__dirname, "dist"),
     },
 }
```

As you can see, we've changed our config object's `entry` field to include two separate Entry Points. One for our existing `index.js` Entry Point, and another for the new `admin.js` one. We've also tweaked the output configuration. Instead of a static name for the resulting output bundle, we use a pattern to describe them. In this case, `[name].js` makes it so we end up with two bundle files named after their corresponding Entry Points. The `[name]` part is where the magic happens. When creating the compiled bundle files, Webpack knows to substitute that "variable" with the corresponding value from the Entry Point configuration.

Go ahead and run `npx webpack --config webpack.config.js` again and inspect the `dist` directory. You will see that we now have two bundles: `index.js` and `admin.js`.

```
dist/
├── admin.js
└── index.js
```

The new `admin.js` file can be added to the new `admin.html` page like usual, via a `<script>` tag. Just like we did with `index.html`.

### Bonus: CSS can also be a module

When discussing Loaders, I mentioned the possibility of making things other than JavaScript behave like modules and be available to the application. Let's demonstrate how we can make that happen.

First, let's install the loaders that we need with:

```
npm install --save-dev css-loader style-loader
```

Then, create a new CSS file under `/css/index.css` with some random fabulous styling:

```css
body {
    background-color: aquamarine;
}

button, input {
    border: grey solid 1px;
    background-color: white;
    padding: 5px;
}

input:hover, button:hover {
    background-color: teal;
}
```

Now "import" it into our `index.js` file with a line like this near the top of the file:

```javascript
import "../css/index.css";
```

And finally, configure Webpack so that it knows what to do when it finds this weird import css line. We do so by updating our `module.rules` config:

```diff
 module: {
     rules: [
         { test: /\.js$/, exclude: /node_modules/, loader: "babel-loader" },
+        { test: /\.css$/, use: ['style-loader', 'css-loader'] }
     ]
 }
```

What we did here was add the new loaders to the Webpack build pipeline. With this, it now knows how to handle CSS files and apply whatever styling rules are defined in that CSS to the page in question. Pretty neat trick, huh?

> These two particular loaders are doing more than meets the eye. You can learn more about them in https://github.com/webpack-contrib/css-loader and https://github.com/webpack-contrib/style-loader.

### Further reading: Plugins

There's one core concept that we haven't discussed yet: Plugins. Plugins offer yet another avenue for further customizing Webpack's behavior. I won't go into too much detail on them here because I think that, with the understanding we have gained on Entry Points, Outputs and Loaders; we've added really powerful tools into our toolboxes. Tools that will be enough in most cases or at least allow us to get any new project up and running. But this time, with more insight into what's actually happening under Webpack's hood. If you are so inclined, you can learn more about them here: https://webpack.js.org/concepts/plugins/. In a nutshell, Plugins are for "doing anything else that a loader cannot do" as Webpack's own documentation puts it.

~

And that's it for now. Thanks for joining me while I explored Webpack, a pretty neat piece of software that makes many a front end developer's life easier. Mine included.