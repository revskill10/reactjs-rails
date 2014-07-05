# Setup project

    rails new railsreact

### Add these lines to Gemfile
    gem 'react-rails'
    group :test do
    	gem 'capybara'
    	gem 'rspec-rails'
    	gem 'poltergeist'
    end
Install gems:

    bundle install
Install rspec:

    rails g rspec:install

Remove `--warnings` in `.rspec`:

    --color
    --require rails_helper


Configure phantomjs with Rails, put these line in `spec/rails_helper`:

    require 'capybara/poltergeist'
    Capybara.javascript_driver = :poltergeist

Create file `phantomjs-shims.js` in `assets/javascript` folder with content  ( [source](https://github.com/facebook/react/blob/master/src/test/phantomjs-shims.js)):

    (function() {

    var Ap = Array.prototype;
    var slice = Ap.slice;
    var Fp = Function.prototype;

    if (!Fp.bind) {
      // PhantomJS doesn't support Function.prototype.bind natively, so
      // polyfill it whenever this module is required.
      Fp.bind = function(context) {
        var func = this;
        var args = slice.call(arguments, 1);

        function bound() {
          var invokedAsConstructor = func.prototype && (this instanceof func);
          return func.apply(
            // Ignore the context parameter when invoking the bound function
            // as a constructor. Note that this includes not only constructor
            // invocations using the new keyword but also calls to base class
            // constructors such as BaseClass.call(this, ...) or super(...).
            !invokedAsConstructor && context || this,
            args.concat(slice.call(arguments))
          );
        }

        // The bound function must share the .prototype of the unbound
        // function so that any object created by one constructor will count
        // as an instance of both constructors.
        bound.prototype = func.prototype;

        return bound;
      };
    }

    })();

Add `phantomjs-shims.js` to `assets/javascripts/application.js`:

    //= require jquery
    //= require jquery_ujs
    //= require turbolinks
    //= require phantomjs-shims
    //= require react

### Install phantomjs
Mac

>Homebrew: `brew install phantomjs`
    MacPorts: `sudo port install phantomjs`
    Manual install: [Download this](https://bitbucket.org/ariya/phantomjs/downloads/phantomjs-1.9.7-macosx.zip)

Linux

> Download the 32 bit or 64 bit binary.
    Extract the tarball and copy bin/phantomjs into your PATH

Windows


> Download the [precompiled binary](https://bitbucket.org/ariya/phantomjs/downloads/phantomjs-1.9.7-windows.zip) for Windows


### Test all:

Create folder `spec/features` with file `create_blog_spec`:

    require 'rails_helper'

    feature 'Creating blog' do
    	scenario 'can create a blog', :js => true do
    		visit '/'
    		click_link 'New Blog'

    		fill_in 'Name', with: 'My blog'
    		click_button 'Create Blog'
    		expect(page).to have_content('Blog successfully created.')

    	end

    end

Now test the spec:

    rspec

If you see the log like this:

    ←[31mF←[0m

    Failures:

      1) Creating blog can create a blog
         ←[31mFailure/Error: fill_in 'Name', with: 'My blog'←[0m
         ←[31mCapybara::ElementNotFound:←[0m
         ←[31m  Unable to find field "Name"←[0m
         ←[36m# ./spec/features/create_blog_spec.rb:8:in `block (2 levels) in <top (
    required)>'←[0m

    Finished in 6.91 seconds (files took 22.74 seconds to load)
    ←[31m1 example, 1 failure←[0m

    Failed examples:

    ←[31mrspec ./spec/features/create_blog_spec.rb:4←[0m ←[36m# Creating blog can cr
    eate a blog←[0m

**You've done! Cheers.**
