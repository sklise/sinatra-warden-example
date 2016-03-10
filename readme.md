# Sinatra Warden Example

_This readme is copied from the original blog post [on my site](http://sklise.com/2013/03/08/sinatra-warden-auth/)._

_UPDATE 5/18/2014, Switched from Rack::Flash to Sinatra/Flash and added instructions for launching the app._

In this article I'll explain the basics of authentication and Rack middleware
and in the process build a complete app with [Sinatra](http://sinatrarb.com),
[DataMapper](http://datamapper.org) and [Warden](http://github.com/hassox/warden).

## Audience

This article is intended for people familiar with Sinatra and DataMapper who want multiple user authentication.

If you've never built a website with Sinatra I'd recommend Peepcode's excellent [Meet Sinatra](https://peepcode.com/products/sinatra) screencast, it is definitely worth the twelve dollars.

## Storing Passwords

Passwords should never be stored in plain text. If someone were to get access to your database they'd have all of the passwords. _You'd_ have everyone's passwords. We need to encrypt the passwords. DataMapper supports a BCryptHash property type which is great because [bcrypt](http://en.wikipedia.org/wiki/Bcrypt) is pretty dang [secure](http://codahale.com/how-to-safely-store-a-password/).

Let's get started on a `User` model. For the rest of this section we will be building a file named `model.rb` in stages. The first step is to install the gems we need:

    $ gem install data_mapper
    $ gem install dm-sqlite-adapter

When installing the `data_mapper` gem `bcrypt-ruby` is installed as a dependency.

*Note: you may need to run the above gem commands with `sudo` if you are not using [rvm](http://rvm.io).*

Open up (or create) a file named model.rb and require the gems and set up DataMapper:

###### /model.rb
~~~ruby
require 'rubygems'
require 'data_mapper'
require 'dm-sqlite-adapter'
require 'bcrypt'

DataMapper.setup(:default, "sqlite://#{Dir.pwd}/db.sqlite")
~~~

Now let's create a User model. In addition to including `DataMapper::Resource` we will include the `BCrypt` class (the gem is named 'bcrypt-ruby', it is required as 'bcrypt' and the class is named `BCrypt`).

###### /model.rb (cont.)
~~~ruby
#...

class User
  include DataMapper::Resource

  property :id, Serial, :key => true
  property :username, String, :length => 3..50
  property :password, BCryptHash
end

DataMapper.finalize
DataMapper.auto_upgrade!

# end of model.rb
~~~

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

*If you'd like to see another take on using bcrypt, Github user **namelessjon** has a more complex example with some discussion [here](https://gist.github.com/namelessjon/1039058).*

## Warden, a Library for Authentication and User Sessions

Warden is an excellent gem for authentication with Sinatra. I've found that the documentation for Warden is lacking which is why I'm writing this. If you want to know the why of Warden [read this](https://github.com/hassox/warden/wiki/overview).

You may have seen that there is a gem called [sinatra_warden](https://github.com/jsmestad/sinatra_warden). Why am I not using that? The sinatra_warden gem chooses the routes for logging in and logging out for you and that logic is buried in the gem. I like for all of the routes in my Sinatra apps to be visible at a glance and not squirreled away.

But ok, on to Warden.

After struggling a lot with figuring out how to set up Warden I found [this post](http://mikeebert.tumblr.com/post/27097231613/wiring-up-warden-sinatra) by [Mike Ebert](https://twitter.com/mikeebert) extremely helpful.

Warden is middleware for [Rack](http://rack.github.com/). Sinatra runs on Rack. Rack is an adapter to let Sinatra run on many different web servers. Warden lives between Rack and Sinatra.

I use `bundler` with Sinatra, [this](https://github.com/sklise/sinatra-warden-example/blob/master/Gemfile) is the Gemfile for this example app. Before You'll need to create that Gemfile in your directory and run the following in Terminal:

    $ bundle install

We're using `sinatra-flash` to show alerts on pages, the first chunk of code will load our gems and create a new Sinatra app and register session support and the flash messages:

###### /app.rb
~~~ruby
require 'bundler'
Bundler.require

# load the Database and User model
require './model'

class SinatraWardenExample < Sinatra::Base
  enable :sessions
  register Sinatra::Flash

#...
~~~

Now in the Warden setup. Most of the lines need to be explained so I'll mark up the code with comments. This block tells Warden how to set up, using some code specific to this example, if your user model is named User and has a key of `id` this block should be the same for you, otherwise, replace where you see User with your model's class name.

###### /app.rb (cont)
~~~ruby
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
    # Because authentication failure can happen on any request but
    # we handle it only under "post '/auth/unauthenticated'", we need
    # to change request to POST
    env['REQUEST_METHOD'] = 'POST'
    # And we need to do the following to work with  Rack::MethodOverride
    env.each do |key, value|
      env[key]['_method'] = 'post' if key == 'rack.request.form_hash'
    end
  end
~~~

The last part of setting up Warden is to write the code for the `:password` strategy we called above. In the following block, they keys of `params` which I am using are based on the login form I made.

###### /app.rb (cont)
~~~ruby
  Warden::Strategies.add(:password) do
    def valid?
      params['user'] && params['user']['username'] && params['user']['password']
    end

    def authenticate!
      user = User.first(username: params['user']['username'])

      if user.nil?
        throw(:warden, message: "The username you entered does not exist.")
      elsif user.authenticate(params['user']['password'])
        success!(user)
      else
        throw(:warden, message: "The username and password combination ")
      end
    end
  end
~~~

Hold on a minute. I called an `authenticate` method on `user`. We need to create such a method in our User class that accepts an attempted password. Back in model.rb we'll add the following:

###### /model.rb (reopened)
~~~ruby
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
~~~

Time to define a few routes to handle logging in, logging out and a protected page.

###### /app.rb (cont)
~~~ruby
  get '/' do
    erb :index
  end

  get '/auth/login' do
    erb :login
  end

  post '/auth/login' do
    env['warden'].authenticate!

    flash[:success] = "Successfully logged in"

    if session[:return_to].nil?
      redirect '/'
    else
      redirect session[:return_to]
    end
  end

  get '/auth/logout' do
    env['warden'].raw_session.inspect
    env['warden'].logout
    flash[:success] = 'Successfully logged out'
    redirect '/'
  end

  post '/auth/unauthenticated' do
    session[:return_to] = env['warden.options'][:attempted_path] if session[:return_to].nil?

    # Set the error and use a fallback if the message is not defined
    flash[:error] = env['warden.options'][:message] || "You must log in"
    redirect '/auth/login'
  end

  get '/protected' do
    env['warden'].authenticate!

    erb :protected
  end
end
~~~

## Starting The App

As @Celandir has pointed out, this app uses the [Sinatra modular-style](http://www.sinatrarb.com/intro.html#Modular%20vs.%20Classic%20Style) app. To run a modular app we use a file named `config.ru` (the "ru" stands for rackup).

There are two ways to run this app.

### rackup

When you've ran `bundle install` you'll get a program named 'rackup' which will run the app on port 9292 by default. You need to run "rackup" with the config.ru file, as such:

~~~bash
$ rackup config.ru
# [2014-05-18 12:11:27] INFO  WEBrick 1.3.1
# [2014-05-18 12:11:27] INFO  ruby 2.0.0 (2014-02-24) [x86_64-darwin13.1.0]
# [2014-05-18 12:11:27] INFO  WEBrick::HTTPServer#start: pid=72027 port=9292
~~~

With that running in Terminal visit http://localhost:9292 to see the app.

### shotgun

There is a ruby gem called **shotgun** which is very useful in development because it will pick up changes to your ruby files. So you won't need to stop and restart the server every time you change a file. To use shotgun with our config.ru file, you need to tell shotgun which file to use, like so:

~~~bash
$ shotgun config.ru
# == Shotgun/Thin on http://127.0.0.1:9393/
# >> Thin web server (v1.4.1 codename Chromeo)
# >> Maximum connections set to 1024
# >> Listening on 127.0.0.1:9393, CTRL+C to stop
~~~

Shotgun runs apps on a different port than rackup, if you are using shotgun visit the app at http://localhost:9393.

#### shotgun and flash messages

The flash plugin makes use of sessions to store messages across routes. The sessions are stored with a "secret" generated each time the server starts. `shotgun` works by restarting the server at every request, which means your flash messages will be lost.

To enable flash messages with `shotgun`, you must specifically set `:session_secret` using the following:

~~~ruby
class SinatraWardenExample < Sinatra::Base
  enable :sessions
  register Sinatra::Flash
  set :session_secret, "supersecret"
#...
~~~

Always be careful with storing secret keys in your source code. In fact, it's advisable to not do so, and instead use an `ENV` variable as such:

~~~ruby
set :session_secret, ENV['SESSION_SECRET']
~~~

I figured this out by reading [this very helpful StackOverflow answer](http://stackoverflow.com/questions/5631862/sinatra-and-session-variables-which-are-not-being-set).
