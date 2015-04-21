# Authentication

Great! You now can create blog, view blog, and show blog.

But the blog has no author. And anyone can create blog. It's not what we want. Let's modify the spec:


> As a guest user, instead of the 'New Blog' link, i should see the 'Sign up' and 'Sign in' link,so that i could sign up and sign in. Once signing in, i would see the 'New Blog' link.

Based on what we've done till now, let's refactor the original spec.

I don't want to do a full reload page after clicking 'Sign in' or 'Sign up' button. One way is to make rails return ajax/html response, which contains the javascript code of React components. OK, let's write the spec:

    require 'rails_helper'

    feature 'Guest can sign in' do 
    	scenario 'Guest can sign in', :js => true do 
    		CreateUser.perform(fullname: 'Truong Dung', email: 'test@example.com', password: 'password')
    		visit '/'
    		click_link 'Sign in'
    		fill_in 'email', :with => 'test@example.com'
    		fill_in 'password', :with => 'password'
    		expect(page).to have_content('Welcome Truong Dung')
    	end
    end

Let's make this spec pass.

The error is: `Uninitialized constant User`. So let's create model `User` by running the generator:

    rails g model User fullname email password_encrypted
    bundle exec rake db:migrate

Run `rspec` again:

    bundle exec rspec
    
The error is : `Uninitialized constant CreateUser`. So let's create it in `app/services/create_user.rb`

    class CreateUser
    	def self.perform(options)
    
    	end
    end
    
Don't forget to add the `services` folder to `autoload_path` in `config/application.rb`:    

    require File.expand_path('../boot', __FILE__)

    require 'rails/all'
    
    # Require the gems listed in Gemfile, including any gems
    # you've limited to :test, :development, or :production.
    Bundler.require(:default, Rails.env)
    
    module Testreact
      class Application < Rails::Application
        # Settings in config/environments/* take precedence over those specified here.
        # Application configuration should go into files in config/initializers
        # -- all .rb files in that directory are automatically loaded.
    
        # Set Time.zone default to the specified zone and make Active Record auto-convert to this zone.
        # Run "rake -D time" for a list of tasks for finding time zone names. Default is UTC.
        # config.time_zone = 'Central Time (US & Canada)'
    
        # The default locale is :en and all translations from config/locales/*.rb,yml are auto loaded.
        # config.i18n.load_path += Dir[Rails.root.join('my', 'locales', '*.{rb,yml}').to_s]
        # config.i18n.default_locale = :de
        config.assets.initialize_on_precompile = true
        config.autoload_paths += %W( #{config.root}/app/services )
      end
    end

Run `rspec` again to see next error:

    Failure/Error: click_link 'Sign in'
    Capybara::ElementNotFound:
     Unable to find link "Sign in"
     
This error says that there is no HTML element to input. Let's create it.

Create file `authentication.jsx` in `app/assets/javascripts`:

    /**
     * @jsx React.DOM
     */
    
     var Authentication = React.createClass({
     	getInitialState: function(){
     		return {};
     	},
     	componentWillMount: function(){
    
     	},
     	render: function(){
     		
     			return (
     				<p>
     					<a href="#/sign_in">Sign in</a> | <a href="#/sign_up">Sign up</a>
     				</p>
     			);
     		}
     	
     });
     
Add it to `app/assets/javascripts/index.js.jsx` file:

    /**
     * @jsx React.DOM
     */
    //= require authentication  
    ...
    var HomeView = React.createClass({
    ...
    render : function() {
      	var x = this.state.blogs.map(function(d){
      		return <li key={"blogs_" + d.id}><a href={"#/blogs/" + d.id}>{d.name}</a></li>
      	});
        return (
        	<div>
          <Authentication />
        	<p>{this.props.message}</p>
        	<a href="#/blogs/new">New Blog</a>
        	<br />
        	<h3>All Blogs:</h3>
        	<ul>{x}</ul>
        	</div>
        );
      }
    ...
    });
    
    