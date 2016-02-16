---
layout: post
comments: true
title: "Rails Feature: Password Reset"
date: 2015-02-10
categories: rails
tags:
  rails
  code
  intermediate
---
I wanted to start a series on my blog that focuses on features that I've
implemented in Rails apps. This series is going to be a personal reference and
hopefully a guide for other developers to use. The format of this series will
start with an overview of the feature then I will go over how I implemented it
with code snippets as well as my reasonings.

<!--break-->

So the first feature I implemented was the ability for users of an app to reset
their password. This was for the Netflix clone app that I've been working on for
[Tealeaf Academy][tealeaf]. This is definitely a feature you'd want users to
have in case they forget their password. Other gems such as [Devise][devise]
have this included but it's always good to create these features from scratch.

Here is a breakdown of how I feel the password reset process should flow:

#### Initial forgot password request:
- Requires user's email address
- If email address is found
  - email with password reset link is sent
  - redirects to a confirmation page
- If email address is not found
  - redirects to input page
  - shows error message that email was not found

#### Password reset email:
- User will follow the provided link
- If it is the user's first time visiting the link
  - asks the user to input a new password
- If it is not the user's first time visiting the link
  - redirects to a page saying that the link has expired

#### Planned Solution:
- Secret tokens that would link users with the password reset request
- The password reset request will expire after the link has been clicked
- Secret token will be set for the user with every password reset request
- Secret token will be deleted after the link from the email has been clicked.

This solution ensures that users have to submit a password reset request every
time. Since the token is deleted after the link is clicked, hackers can't use
the original link and its secret token to gain access to the user's information.

#### Implementation:

I decided to create a new controller to manage these actions and named it
`ResetPasswordController`. To follow the RESTful design my `routes.rb` file
looked like the following:

``` ruby
get 'forget_password', to: 'reset_password#new'
post 'forget_password', to: 'reset_password#create'
get 'reset_password', to: 'reset_password#edit'
patch 'reset_password', to: 'reset_password#update'
```

Next I needed to add a column to the `users` table to store the secret tokens.

`rails generate migration add_secret_token_to_users`

My migration file looked like the following:

``` ruby
class AddSecretTokenToUsers < ActiveRecord::Migration
  def change
    add_column :users, :secret_token, :string
  end
end
```

With everything setup, I started adding the actions to the
`ResetPasswordController`. The first was displaying the forgot password form,
which asks the user for their email address.

All the new action needs to do is render the view.

**NEW action:**

Now this explicit definition of the action is not necessary if the view is
present but it's good practice to show it.

`reset_password_controller.rb`

``` ruby
class ResetPasswordController < ApplicationController
  def new
  end
end
```

For the view I just needed a simple form for the user to enter their email
address. I used a `form_tag` and directed it to the `forget_password` path.
I used Bootstrap to make it look prettier.

`new.html.haml`

``` haml
=form_tag 'forgot_password' do
  %h1 Forgot Password?
  %p We will send you an email with a link that you can use to reset your
  password.
  .form-group
    %label Email Address
    .row
      .col-sm-4
        =text_field_tag :email, nil, class: 'form-control'
  .form-group.action
    =submit_tag 'Send Email', class: 'btn btn-default'
```

**CREATE action:**

Next action was handling the submission of the forgot password request with the
`create` action. We need to check that a user exists with the email address
submitted. If a user doesn't exist, it will redirect to the forgot password
request form again with an error message. If a user does exist they will be
emailed the password reset email and redirected to a confirmation page. I needed
to add this endpoint to my routes and add the action to my controller.

`routes.rb`

``` ruby
get 'confirm_password_reset', to: 'reset_password#confirm'
```

`reset_password_controller.rb`

``` ruby
class ResetPasswordController < ApplicationController
  def create
    @user = User.find_by(email: params[:email])
    if @user
      @user.secret_token = SecureRandom.urlsafe_base64
      @user.save
      UserMailer.reset_password_email(@user).deliver
      redirect_to confirm_path
    else
      flash[:error] = "Could not find email address."
      redirect_to forgot_password_path
    end
  end

  def confirm
  end
end
```

There is a really neat class that Ruby has, which I used to generate the
secret tokens for users. It is called `SecureRandom` and it has several methods
that generate a random string. I used the `urlsafe_base64` method to generate
the secret token because it would be included in the reset password url as a
parameter. You can read more on the class and its methods [here][urlsafe].

As you can also see I included a `UserMailer` method called
`reset_password_email`. I won't go over how to user Rail's ActionMailer but if
you'd like more information, [RailsGuides][actionmailer] has a good introduction
guide on it. I will show you what the `reset_password_email` method looked like.

`user_mailer.rb`

``` ruby
class UserMailer < ActionMailer::Base
  def reset_password_email(user)
    @user = user
    @url = reset_password_url(t: user.secret_token)
    mail from: "donotreply@example.com", to: user.email, subject: "Password
    Reset"
  end
end
```

Rails has this awesome method to reference endpoints from external requests but
changing the `_path` suffix to `_url`. I also added a `t` parameter that would
include the user's secret token. I named the parameter `t` to mask its purpose.
I used this to identify the correct user that requested the password reset.

**EDIT action:**

Now I had to handle the request from the reset password email. I thought the
`edit` action would be a good place since the user was changing their password.
I wanted to make sure the reset password link only could be clicked once. To
accomplish this I would give the user a new secret token every time the `edit`
action was called. This ensured that the reset password secret token would only
match the user for the first click of the link. If the link was already clicked
then the user would be redirected to a page indicating that the link had
expired.

`reset_password_controller.rb`

``` ruby
class ResetPasswordController < ApplicationController
  def edit
    @user = User.find_by(secre_token: params[:t])
    if @user
      @new_token = SecureRandom.urlsafe_base64
      @user.secret_token = @new_token
      @user.save
    else
      redirect_to link_expired_path
    end
  end

  def link_expired
  end
end
```

The view for `edit` would be a simple form asking for the new password. The
important thing to include is a hidden field that has a value of the user's
new secret token to identify which user's password is being reset. Also make sure
the form is submitted using a `patch` method to route correctly.

`edit.html.haml`

``` haml
=form_tag 'reset_password', method: 'patch' do
  %h1 Reset Password
  %p We will send you an email with a link that you can use to reset your
  password.
  .form-group
    %label New Password
    .row
      .col-sm-4
        =hidden_field_tag :t, @new_token
        =text_field_tag :password, nil, class: 'form-control'
  .form-group.action
    =submit_tag 'Reset Password', class: 'btn btn-default'
```

**UPDATE action:**

In order to handle the form submission I had to find the correct user and thanks
to the secret token, this was possible. If a user was found then I would update
the user's password with the new one. They would be redirected to the sign in
with a success message. If a user was not found they would be redirected to the
link expired page. If everything works correctly the condition should never be
met, but in case of some type of outside request, it would be able to handle it.

`reset_password_controller.rb`

``` ruby
class ResetPasswordController < ApplicationController
  def update
    @user = User.find_by(secre_token: params[:t])
    if @user
      @user.password = params[:password]
      @user.secret_token = nil
      @user.save
      redirect_to signin_path, notice: "Password was successfully reset. Please
      sign in again."
    else
      redirect_to link_expired_path
    end
  end
end
```

At this point the feature worked for me the way I wanted it to. As always I also
created some Rspec tests but I will not go over them in this post. I apologize
for the lengthiness of this post but hopefully you got some useful information
from it! As always thanks for reading, please leave any feedback and happy
coding!


[tealeaf]: https://www.gotealeaf.com
[devise]: https://github.com/plataformatec/devise
[haml]: http://haml.info
[urlsafe]:
http://www.ruby-doc.org/stdlib-1.9.3/libdoc/securerandom/rdoc/SecureRandom.html#method-c-urlsafe_base64
[actionmailer]: http://guides.rubyonrails.org/action_mailer_basics.html
