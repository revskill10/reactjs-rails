# Second spec: View blog

> When i visit home page, i could click on one blog, and see all entries in the blog.


    require 'rails_helper'

    feature 'guest can view blog' do
    	scenario 'guest can view blog', :js => true do
    		b = Blog.create(name: 'My first blog')
    		b.entries.create(title: 'First entry', content: 'React.js is awesome')
    		visit '/'
    		click_link 'My first blog'
    		expect(page).to have_content 'First entry'
    		expect(page).to have_content 'React.js is awesome'
    	end
    end

There are some errors when you run this test because:
- There is no method entries for Blog instance.
- There is nothing to show when click on blog path

Let's fix them.

Firstly, we need to use `gem database_cleaner` to use database truncation strategy. The reason is your javascript call will open new transaction to database, but the default transactional strategy of Rspec will keep data in all transactions independent. That's why you will se no data in a javascript interaction session. So let's add to `Gemfile`, in `group: :test`

    gem 'database_cleaner'

And install it:

    bundle install
We also configure `Rspec` to use `database_cleaner:`

The `spec/rails_helper.rb` now have the following content:

        # This file is copied to spec/ when you run 'rails generate rspec:install'
    ENV["RAILS_ENV"] ||= 'test'
    require 'spec_helper'
    require File.expand_path("../../config/environment", __FILE__)
    require 'rspec/rails'
    require 'capybara/poltergeist'
    Capybara.javascript_driver = :poltergeist
    require 'database_cleaner'
    # Requires supporting ruby files with custom matchers and macros, etc, in
    # spec/support/ and its subdirectories. Files matching `spec/**/*_spec.rb` are
    # run as spec files by default. This means that files in spec/support that end
    # in _spec.rb will both be required and run as specs, causing the specs to be
    # run twice. It is recommended that you do not name files matching this glob to
    # end with _spec.rb. You can configure this pattern with with the --pattern
    # option on the command line or in ~/.rspec, .rspec or `.rspec-local`.
    Dir[Rails.root.join("spec/support/**/*.rb")].each { |f| require f }

    # Checks for pending migrations before tests are run.
    # If you are not using ActiveRecord, you can remove this line.
    ActiveRecord::Migration.check_pending! if defined?(ActiveRecord::Migration)

    RSpec.configure do |config|
      # Remove this line if you're not using ActiveRecord or ActiveRecord fixtures
      config.fixture_path = "#{::Rails.root}/spec/fixtures"

      # If you're not using ActiveRecord, or you'd prefer not to run each of your
      # examples within a transaction, remove the following line or assign false
      # instead of true.
      #config.use_transactional_fixtures = true
      config.use_transactional_fixtures = false

      config.before(:suite) do
        DatabaseCleaner.strategy = :truncation
      end

      config.before(:each) do
        DatabaseCleaner.start
      end

      config.after(:each) do
        DatabaseCleaner.clean
      end

      # RSpec Rails can automatically mix in different behaviours to your tests
      # based on their file location, for example enabling you to call `get` and
      # `post` in specs under `spec/controllers`.
      #
      # You can disable this behaviour by removing the line below, and instead
      # explicitly tag your specs with their type, e.g.:
      #
      #     RSpec.describe UsersController, :type => :controller do
      #       # ...
      #     end
      #
      # The different available types are documented in the features, such as in
      # https://relishapp.com/rspec/rspec-rails/docs
      config.infer_spec_type_from_file_location!
    end

Let's generate model:

    rails g model Entry title:string content:text blog_id:integer
    rake db:migrate
    rake db:test:clone


`app/models/blog.rb:`

    class Blog < ActiveRecord::Base
    	has_many :entries, :dependent => :destroy
    	def as_json(options={})
    	  super(:only => [:id, :name], :include => :entries)
    	end
    end


`app/models/entry.rb:`

    class Entry < ActiveRecord::Base
    	belongs_to :blog
    	def as_json(options={})
    	  super(:only => [:title, :content])
    	end
    end


`config/routes.rb:`

    resources :blogs do
        resources :entries
    end

Add `BlogView` in `assets/javascripts/index.js.jsx:`

    var BlogView = React.createClass({
      getInitialState: function(){
        return {name: '', entries: []}
      },
      componentWillMount: function(){
        $.ajax({
          url: '/blogs/' + this.props.blog_id,
          dataType: 'json',
          type: 'GET',
          success: function(data){
            this.setState({name: data.name, entries: data.entries});
          }.bind(this)
        });
      },
      render: function(){
        var x = this.state.entries.map(function(d){
          if (d != null) {
            return <li key={d.title}>
              <p>{d.title}</p>
              <p>{d.content}</p>
            </li>
          }
        })
        return (
          <div>
          <h1>{this.state.name}</h1>
          <ul>
            {x}
          </ul>
          </div>
        )
      }
    });

The `Router` class now have the code like that:

    var Router = Backbone.Router.extend({
      message: '',
      routes : {
        "" : "index",
        "blogs/new" : "new_blog",
        "blogs/:blog_id": "view_blog"
      },
      index : function() {
      	var self = this;
        React.renderComponent(
          <HomeView message={self.message}/>,
          document.getElementById('new-blog')
        );
      },
      new_blog : function() {
        React.renderComponent(
          <NewBlogView message={self.message}/>,
          document.getElementById('new-blog')
        );
      },
      view_blog: function(blog_id){
        React.renderComponent(
          <BlogView blog_id={blog_id}/>,
          document.getElementById('new-blog')
        )
      }
    });
And lastly, add method `show` to controller `app/controllers/blogs_controller.rb:`

    def show
        @blog = Blog.find(params[:id])
        render json: @blog
    end


The test should pass now.
