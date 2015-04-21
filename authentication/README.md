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

The error is: `Uninitialized constant User`. So let's create model `User` in `models/User.rb`:

    class User < ActiveRecord::Base
    
    end