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
