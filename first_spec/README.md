# First spec

> When i visit homepage, i should see the `new blog`link

#### `spec/features/create_blog_spec.rb:`

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

This test failed. Because:
- You don't have any root route.
- You don't have any view with the 'New Blog' link.
- You don't have any form.

Let's make this pass:

#### `config/routes.rb:`

    root 'home#index'
    resources :blogs

#### Generate blog model:

    rails g controller Home index
    rails g model Blog name
    rails g controller Blogs new create

#### `views/home/index.html.erb:`

    <div id='new-blog'></div>
    <script type="text/javascript" src="/assets/index.js"></script>

#### Create file `index.js.jsx` in `assets/javascripts` folder:

    /**
     * @jsx React.DOM
     */

    // Set up token to send when submit form to rails via ajax.
    var token = $( 'meta[name="csrf-token"]' ).attr( 'content' );

    $.ajaxSetup( {
    	beforeSend: function ( xhr ) {
    		xhr.setRequestHeader( 'X-CSRF-Token', token );
    	}
    });
    // this.props.message is the flash message returned by rails.
    var HomeView = React.createClass({
      render : function() {
        return (
        	<div>
        	<p>{this.props.message}</p>
        	<a href="#/blogs/new">New Blog</a>
        	</div>
        );
      }
    });
    // This view is simply a form with submit the json parameters and receive the flash message
    var NewBlogView = React.createClass({
    	handleSubmit: function(){
    		var name = this.refs.name.getDOMNode().value.trim();
    		$.ajax({
    	      url: '/blogs',
    	      dataType: 'json',
    	      type: 'POST',
    	      data: {
    	      	'blog': {
    	      		'name': name
    	      	}
    	      },
    	      success: function(data) {
    	        r.message = data.message;
    	        r.navigate('/', {trigger: true});
    	      }.bind(this)
    	    });
    		return false;
    	},
      render : function() {
        return (
        	<div>
        	<form onSubmit={this.handleSubmit}>
        		<label>Name</label><br/>
        		<input type="text" name="name" id="name" ref="name" /><br/>
        		<input type="submit" value="Create Blog" />
        	</form>
        	<a href="/">Home</a>
        	</div>
        );
      }
    });


    var Router = Backbone.Router.extend({
      message: '',
      routes : {
        "" : "index",
        "blogs/new" : "new_blog"
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
      }
    });

    var r = new Router();

    Backbone.history.start();

#### `controllers/blogs_controller.rb:`

    class BlogsController < ApplicationController
      def new
      end

      def create
      	blog = Blog.new(blog_params)
      	if blog.save
      		flash[:notice] = 'Blog successfully created.'
      		render json: {:message => flash[:notice]}
      	else
      		flash[:error] = 'Error'
      		render json: flash
      	end
      end
      private
      def blog_params
      	params.require(:blog).permit(:name)
      end
    end


The test should pass.
