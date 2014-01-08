# Sinatra Warden Example

_This readme is copied from the original blog post [on my site](http://skli.se/2013/03/08/sinatra-warden-auth/)._

In this article I'll explain the basics of authentication and Rack middleware
and in the process build a complete app with [Sinatra](http://sinatrarb.com),
[DataMapper](http://datamapper.org) and [Warden](http://github.com/hassox/warden).

## Audience

This article is intended for people familiar with Sinatra and DataMapper who
want multiple user authentication.

If you've never built a website with Sinatra I'd recommend Peepcode's excellent
[Meet Sinatra](https://peepcode.com/products/sinatra) screencast, it is
definitely worth the twelve dollars.

## Storing Passwords

Passwords should never be stored in plain text. If someone were to get access
to your database they'd have all of the passwords. _You'd_ have everyone's
passwords. We need to encrypt the passwords. DataMapper supports a BCryptHash
property type which is great because [bcrypt](http://en.wikipedia.org/wiki/Bcrypt) is pretty dang
[secure](http://codahale.com/how-to-safely-store-a-password/).

Let's get started on a `User` model. For the rest of this section we will be
building a file named `model.rb` in stages. The first step is to install the
gems we need:

    $ gem install data_mapper
    $ gem install dm-sqlite-adapter

When installing the `data_mapper` gem `bcrypt-ruby` is installed as a
dependency.

*Note: you may need to run the above gem commands with `sudo` if you are not
using [rvm](http://rvm.io).*

Open up (or create) a file named model.rb and require the gems and set up
DataMapper:

###### /model.rb
```ruby
require 'rubygems'
require 'data_mapper'
require 'dm-sqlite-adapter'
require 'bcrypt'

DataMapper.setup(:default, "sqlite://#{Dir.pwd}/db.sqlite")
```

Now let's create a User model. In addition to including `DataMapper::Resource`
we will include the `BCrypt` class (the gem is named 'bcrypt-ruby', it is
required as 'bcrypt' and the class is named `BCrypt`).

###### /model.rb (cont.)
```ruby
#...

class User
  include DataMapper::Resource
  include BCrypt

  property :id, Serial, :key => true
  property :username, String, :length => 3..50
  property :password, BCryptHash
end

DataMapper.finalize
DataMapper.auto_upgrade!

# end of model.rb
```

Let's test this code.

    $ irb
    > require './model'
    > @user = User.new(:username => "admin", :password => "test")
    > @user.save
    > @user.password
    # => "$2a$10$lKgran7g.1rSYY0M6d0V9.uLInljHgYmrr68LAj86rllmApBSqu0S"
    > @user.password == 'test'
    # => true
    > @user.password
    # => "$2a$10$lKgran7g.1rSYY0M6d0V9.uLInljHgYmrr68LAj86rllmApBSqu0S"
    > exit

Excellent. We have a User model that stores passwords in an encrypted way.

*If you'd like to see another take on using bcrypt, Github user **namelessjon**
has a more complex example with some discussion [here](https://gist.github.com/namelessjon/1039058).*

## Warden, a Library for Authentication and User Sessions

Warden is an excellent gem for authentication with Sinatra. I've found that the
documentation for Warden is lacking which is why I'm writing this. If you want
to know the why of Warden [read this](https://github.com/hassox/warden/wiki
/overview).

You may have seen that there is a gem called [sinatra_warden](https://github.
com/jsmestad/sinatra_warden). Why am I not using that? The sinatra_warden gem
chooses the routes for logging in and logging out for you and that logic is
buried in the gem. I like for all of the routes in my Sinatra apps to be
visible at a glance and not squirreled away.

But ok, on to Warden.

After struggling a lot with figuring out how to set up Warden I found [this
post](http://mikeebert.tumblr.com/post/27097231613/wiring-up-warden-sinatra) by
[Mike Ebert](https://twitter.com/mikeebert) extremely helpful.

Warden is middleware for [Rack](http://rack.github.com/. Sinatra runs on Rack.
Rack is an adapter to let Sinatra run on many different web servers. Warden
lives between Rack and Sinatra.

I use `bundler` with Sinatra, [this](https://github.com/stevenklise/sinatra-warden-example/blob/master/Gemfile) is the Gemfile for this example app. Before
You'll need to create that Gemfile in your directory and run the following in
Terminal:

    $ bundle install

We're using `rack-flash3` to show alerts on pages, the first chunk of code will
load our gems and create a new Sinatra app and register session support and
the flash messages:

###### /app.rb
```ruby
require 'bundler'
Bundler.require

# load the Database and User model
require './model'

class SinatraWardenExample < Sinatra::Base
  use Rack::Session::Cookie, secret: "nothingissecretontheinternet"
  use Rack::Flash, accessorize: [:error, :success]

#...
```

Now in the Warden setup. Most of the lines need to be explained so I'll mark up
the code with comments. This block tells Warden how to set up, using some code
specific to this example, if your user model is named User and has a key of
`id` this block should be the same for you, otherwise, replace where you see
User with your model's class name.

###### /app.rb (cont)
```ruby
  use Warden::Manager do |config|
    # Tell Warden how to save our User info into a session.
    # Sessions can only take strings, not Ruby code, we'll store
    # the User's `id`
    config.serialize_into_session{|user| user.id }
    # Now tell Warden how to take what we've stored in the session
    # and get a User from that information.
    config.serialize_from_session{|id| User.get(id) }

    config.scope_defaults :default,
      # "strategies" is an array of named methods with which to
      # attempt authentication. We have to define this later.
      strategies: [:password],
      # The action is a route to send the user to when
      # warden.authenticate! returns a false answer. We'll show
      # this route below.
      action: 'auth/unauthenticated'
    # When a user tries to log in and cannot, this specifies the
    # app to send the user to.
    config.failure_app = self
  end

  Warden::Manager.before_failure do |env,opts|
    env['REQUEST_METHOD'] = 'POST'
  end
```

The last part of setting up Warden is to write the code for the `:password`
strategy we called above. In the following block, they keys of `params` which I
am using are based on the login form I made.

###### /app.rb (cont)
```ruby
  Warden::Strategies.add(:password) do
    def valid?
      params['user']['username'] && params['user']['password']
    end

    def authenticate!
      user = User.first(username: params['user']['username'])

      if user.nil?
        fail!("The username you entered does not exist.")
        flash.error = ""
      elsif user.authenticate(params['user']['password'])
        flash.success = "Successfully Logged In"
        success!(user)
      else
        fail!("Could not log in")
      end
    end
  end
```

Hold on a minute. I called an `authenticate` method on `user`. We need to
create such a method in our User class that accepts an attempted password. Back
in model.rb we'll add the following:

###### /model.rb (reopened)
```ruby
class User
  #...

  def authenticate(attempted_password)
    if self.password == attempted_password
      true
    else
      false
    end
  end
end
```

Time to define a few routes to handle logging in, logging out and a protected
page.

###### /app.rb (cont)
```ruby
  get '/' do
    erb :index
  end

  get '/auth/login' do
    erb :login
  end

  post '/auth/login' do
    env['warden'].authenticate!

    flash.success = env['warden'].message

    if session[:return_to].nil?
      redirect '/'
    else
      redirect session[:return_to]
    end
  end

  get '/auth/logout' do
    env['warden'].raw_session.inspect
    env['warden'].logout
    flash.success = 'Successfully logged out'
    redirect '/'
  end

  post '/auth/unauthenticated' do
    session[:return_to] = env['warden.options'][:attempted_path]
    puts env['warden.options'][:attempted_path]
    flash.error = env['warden'].message || "You must log in"
    redirect '/auth/login'
  end

  get '/protected' do
    env['warden'].authenticate!
    @current_user = env['warden'].user
    erb :protected
  end
end
```

The code now is getting a bit long for a blog post. And all of the tricky parts
have been detailed. You can download and try out the full app on Github in my
[sinatra-warden-example](http://github.com/stevenklise/sinatra-warden-example).
