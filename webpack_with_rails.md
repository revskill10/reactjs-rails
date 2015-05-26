# Webpack with Rails

The Asset Pipeline (Sprockets) is the canonical way of packaging assets in Rails, it makes things very easy for us as it provides a very clear way on how to load our assets and how to bundle them for production.

I believe the Asset Pipeline is a good approach for most Rails applications, but there are times when it makes things harder and shows limitations.

In this post I want to explore why you should consider using an alternative (Webpack) for bundling your assets for some projects.

## Issues with the Assets Pipeline

**Modularity**

Using a modular approach we would build a large application using small modules. Each module would know and require what it needs in order to function correctly.

In an ideal world we should only load / bundle the code we need for a particular part of an application and no more.

It is possible to build complex dependency trees with the Assets Pipeline thus carefully controlling what is being bundled, but in practice I never see this, most applications just load everything into a few big JS / CSS files.

A typical Sprockets manifest looks like:

    //= require jquery
    //= require jquery_ujs
    //= require moment
    //= require_tree ./models
    //= require_tree ./controllers
...

The fact that you can do require_tree in Sprockets is an indication of its philosophy, which is just to bundle large collections of files without thinking too much about the dependencies between them.

Too often the Sprockets manifests become a dumping ground for code that is used and code that isn't. Over time this file keeps growing as we dump more things in it. It is also hard to figure out what code to take out from it, as it is unclear if it is used and where.
Different entry points

Some Rails applications become so large that loading all of the assets all of the time can be an unnecessary burden for our users. The solution is to have multiple entry points for different parts of our application, each bundle only having the necessary assets for that particular part.

This is possible with Sprockets but in practice it is not clean and elegant, there is a big overhead in getting the dependencies right for each bundle.
CSS and JS have their own pipelines

In the Asset Pipeline there is no way to specify dependencies between your html templates, JavaScript code and CSS. Each of these is a separate pipeline, which makes it difficult and cumbersome to ensure that each component has access to the necessary assets.
The Asset Pipeline solution: bundle all of the world

Because of all these things, the easiest and sanest way to do things with the Asset Pipeline is to load all of the things all of the time. In this way we know that what we need is there and we don't have to micromanage the dependencies. This is ok for many applications but there are better ways.
So what is a good system?

A good system is one where we can simply declare the dependencies for each component / module and have a system that bundles exactly what we need (for any given part of the application). These dependencies could be JavaScript code, Images, CSS, SASS, fonts, templates, etc.

## Webpack

Webpack is such a system, it is a complete alternative to the Asset Pipeline. It lets you declare all of your dependencies in each component (JS, CSS, templates, fonts, etc).

In this way you can build complex dependency trees and Webpack will package them for you only adding the necessary assets.

For example a webpack component might look like:

    require('../component.less');
    var Foo         = require('ap/shared/foo');
    var template    = require('raw!./tmpl.mustache');
    
    module.exports = Foo.extend({
        template: template,
        ...
    });

The important things to notice above is that we are requiring just what we need for a particular component:

    some CSS (component.less)
    some JS code (shared/foo)
    and a mustache template (tpml.mustache).

Then we use those things and export the component to be used by another part of our application (using module.exports).

This example uses the CommonJS syntax, but you can use others like AMD or ES6.

Webpack encourages us to build our application in a modular way, where each component requires what it needs and nothing else.

### Again, why do this?

By building small modules like this we gain a few abilities:

- We can require and bundle only the code that we know we need (JS, CSS, templates) in a maintainable way.
- It becomes very simple to have multiple entry points for an application (with all the right code for each).
- The front-end is portable, we could easily swap the backend to something different than Rails and use it as is.

## Webpack with Rails

To use Webpack with Rails what we need to do is simply forget about the Asset Pipeline and manage all our front-end using the tools that Webpack provides. Webpack is specifically suitable for projects that are JavaScript heavy e.g. Backbone, CanJS, React.

The following is the setup that I am currently using for a project:

### Installing Webpack

Webpack is installed via NPM:

    npm install webpack --save-dev

### Front-end code

In the root of my project I have a folder called fe (front-end). All my front-end code goes here. Not in app/assets/...

A side effect of doing this is that the front-end code becomes a first class citizen in the application, instead of being buried inside app/assets/javascripts/....

In this folder I have my components e.g. fe/clients/index.js. Each component requires what it needs using the CommonJS syntax. e.g.

    var can       = require('can');
    var template  = require('raw!./tmpl.stache');
    var Client    = require('models/client');
    
    module.exports = can.Component.extend({
        template: can.stache(template),
        ...
    });

The whole front end is built like this.
### Configuration

In the root of my project I have a webpack.config.js file that holds the configuration for Webpack, it looks like this:

    var path = require("path");
    
    module.exports = {
        context: __dirname,
        entry: {
            clients:  "./fe/ap/clients/entry.js",
            invoices: "./fe/ap/invoices/entry.js",
        },
        output: {
            path: path.join(__dirname, 'app', 'assets', 'javascripts'),
            filename: "[name]-bundle.js",
            publicPath: "/js/"
        },
        module: {
            loaders: [
                { test: /\.less$/,   loader: "style-loader!css-loader!less-loader"},
                { test: /\.woff$/,   loader: "url-loader?prefix=font/&limit=5000" },
            ]
        },
        resolve: {
            alias: {
                ap:     path.join(__dirname, "fe", "ap"),
                shared: path.join(__dirname, "fe", "ap", "shared"),
    
        }
    };

Notable points are:

- The entry key defines entry points for my application. I can have as many as I want, Webpack will figure out the dependencies for each and build different bundles.
- The output key defines where I want my bundles, in this case I save them in app/assets/javascripts. There they can be easily found by our Rails application.
- loaders define loaders that we use in our modules, e.g. less, sass, fonts, css, etc. Loaders are installed using NPM.

### Building assets

Webpack has a CLI tool that watches for file changes and builds our assets very fast. With it there is no need to learn or use tools like Grunt, Gulp or Broccoli. While developing, I simply run:

    webpack --watch

You can use foreman to make this easier by starting a Rails server and the Webpack watcher with one command.
Loading the modules in Rails views

With this configuration Webpack will build the bundles and put them in app/assets/javascripts. This makes them available to the Rails Asset Pipeline.

We want this so the bundles get fingerprinted during deployment and avoid caching issues this way.

I then have a Sprockets manifest for each Webpack bundle. For example I have clients-bundle.js and invoices-bundle.js, I have two manifests clients.js and invoices.js. These manifest files simply require the Webpack bundles, for example in app/assets/javascripts/bundle.js:

    //= require clients-bundle

Then the views require the manifest files, for example in app/views/clients/show.html.erb:

    <div id="app">
        <%= javascript_include_tag('clients') %>
    </div>

Finally, don't forget to tell Rails to compile these files when deploying. In /config/initializers/assets.rb:

    Rails.application.config.assets.precompile += %w( 
        clients.js
        invoices.js
        ...
    )

## An alternative solution

Instead of using the Asset Pipeline, we could just build the bundles with Webpack and put them somewhere in public. Having Webpack taking care of fingerprinting the bundles. An alternative Webpack config could look like:

    output: {
        path: path.join(__dirname, 'public', 'js'),
        filename: "[name]-bundle-[hash].js",
    },

Note the [hash] parameter, this tell Webpack to add a hash to each generated bundle, making the name unique. I have been experimenting with this approach and created a plug-in that saves a JSON file with the hashes generated by Webpack, then my views read this json file and require the correct file. For more details see.
### To commit or not

When I am working alone in a project I like to keep it simple and commit the generated assets to source control. However this is not a great strategy when working on a team, because of potential merge conflicts. In a team environment it is better not to commit the bundles and only generate them in CI and deployment.
## An hybrid approach

But not necessarily everything needs to go through Webpack. For example we might have libraries that we know are used everywhere in our application e.g. Moment.js, jQuery, React, etc. So we can have a hybrid approach by using the Rails asset pipeline and Webpack.

For example we could have a Sprockets manifest that loads the global libraries using the Asset Pipeline:

    //= require jquery
    //= require jquery_ujs
    //= require lodash
    //= require bootstrap
    //= require loglevel
    //= require toastr
    //= require moment

And load all of our application code using Webpack as shown before.
Conclusion

Webpack offers an interesting alternative to the Rails Asset pipeline with better modularisation and dependency loading. I believe the Asset Pipeline is still a solid way of doing things, but Webpack is worth considering for heavy JavaScript applications with Rails.