# Refactor first spec

> Guest can see all blogs

Create file `spec\guest_see_blogs_spec:`

    require 'rails_helper'

    feature 'Guest can see all blogs' do
    	scenario 'Guest can see all blogs', :js => true do
    		Blog.create('My blog')
    		visit '/'
    		expect(page).to have_content('My blog')
    	end

    end

Add index action in `blogs_controller.rb:`

    def index
        @blogs = Blog.all
        render json: @blogs
    end

Modify `HomeView` in `index.js.jsx`:

    var HomeView = React.createClass({
      getInitialState: function(){
    	return {blogs: []}
      },
      componentWillMount: function(){
      	this.loadBlogs();
      },
      loadBlogs: function(){
      	$.ajax({
      		url: '/blogs',
      		dataType: 'json',
      		type: 'GET',
      		success: function(data){
      			this.setState({blogs: data});
      		}.bind(this)
      	})
      },
      render : function() {
      	var x = this.state.blogs.map(function(d){
      		return <li key={"blogs_" + d.id}><a href={"#/blogs/" + d.id}>{d.name}</a></li>
      	});
        return (
        	<div>
        	<p>{this.props.message}</p>
        	<a href="#/blogs/new">New Blog</a>
        	<br />
        	<h3>All Blogs:</h3>
        	<ul>{x}</ul>
        	</div>
        );
      }
    });

The code here is straightforward. You just reload all blogs whenever the component is rendered.

The test should pass now:

    Finished in 4.82 seconds (files took 10.09 seconds to load)
    ←[32m2 examples, 0 failures←[0m


