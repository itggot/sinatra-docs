Getting Started
===============

Sinatra is a [DSL](https://en.wikipedia.org/wiki/Domain-specific_language) for quickly creating web applications in Ruby with minimal effort:

```ruby
# app.rb
require 'sinatra'

get('/') do
    'Hello world!'
end
``` 

Install the gem:

```bash
gem install sinatra
```
    

And run with:
```bash
ruby app.rb
```

View at: [http://localhost:4567](http://localhost:4567/)

It is recommended to also run `gem install thin`, which Sinatra will pick up if available.

Routes
------

In Sinatra, a route is an HTTP method paired with a URL-matching pattern. Each route is associated with a block:

```ruby
get('/') do
    # show something ..
end

post('/') do
    # change something ..
end
``` 

Routes are matched in the order they are defined. The first route that matches the request is invoked.

Routes with trailing slashes are different from the ones without:

```ruby
get('/foo') do
    # Does not match "GET /foo/"
end
```
  
Dynamic Routes
--------------

Route patterns may include named parameters, accessible via the `params` hash:

```ruby
get('/hello/:name') do
  # matches "GET /hello/foo" and "GET /hello/bar"
  # params['name'] is 'foo' or 'bar'
  "Hello #{params['name']}!"
end
```
    

You can also access named parameters via block parameters:

```ruby
get('/hello/:name') do |n|
  # matches "GET /hello/foo" and "GET /hello/bar"
  # params['name'] is 'foo' or 'bar'
  # n stores params['name']
  "Hello #{n}!"
end
```
    

Route patterns may also include splat (or wildcard) parameters, accessible via the `params['splat']` array:

```ruby
get('/say/*/to/*') do
  # matches /say/hello/to/world
  params['splat'] # => ["hello", "world"]
end

get('/download/*.*') do
  # matches /download/path/to/file.xml
  params['splat'] # => ["path/to/file", "xml"]
end
```

Or with block parameters:

```ruby
get('/download/*.*') do |path, ext|
  [path, ext] # => ["path/to/file", "xml"]
end
``` 

Route matching with Regular Expressions:

```ruby
get(/\/hello\/([\w]+)/)do
  "Hello, #{params['captures'].first}!"
end
```
    
Route patterns may have optional parameters:

```ruby
get('/posts/:format?') do
  # matches "GET /posts/" and any extension "GET /posts/json", "GET /posts/xml" etc
end
```
    

Routes may also utilize query parameters:

```ruby
get('/posts') do
  # matches "GET /posts?title=foo&author=bar"
  title = params['title']
  author = params['author']
  # uses title and author variables; query is optional to the /posts route
end
```
    

By the way, unless you disable the path traversal attack protection (see below), the request path might be modified before matching against your routes.

You may customize the Mustermann options used for a given route by passing in a `:mustermann_opts` hash:

    get '\A/posts\z', :mustermann_opts => { :type => :regexp, :check_anchors => false } do
      # matches /posts exactly, with explicit anchoring
      "If you match an anchored pattern clap your hands!"
    end
    

It looks like a [condition](http://sinatrarb.com/intro.html#conditions), but it isn’t one! These options will be merged into the global `:mustermann_opts` hash described [below](http://sinatrarb.com/intro.html#available-settings).


Return Values
-------------

The return value of a route block determines at least the response body passed on to the HTTP client, or at least the next middleware in the Rack stack. Most commonly, this is a string, as in the above examples. But other values are also accepted.

You can return any object that would either be a valid Rack response, Rack body object or HTTP status code:

*   An Array with three elements: `[status (Fixnum), headers (Hash), response body (responds to #each)]`
*   An Array with two elements: `[status (Fixnum), response body (responds to #each)]`
*   An object that responds to `#each` and passes nothing but strings to the given block
*   A Fixnum representing the status code

---------------------

Static Files
------------

Static files are served from the `./public` directory. You can specify a different location by setting the `:public_folder` option:

    set :public_folder, File.dirname(__FILE__) + '/static'
    

Note that the public directory name is not included in the URL. A file `./public/css/style.css` is made available as `http://example.com/css/style.css`.

Use the `:static_cache_control` setting (see below) to add `Cache-Control` header info.

Views / Templates
-----------------

Each template language is exposed via its own rendering method. These methods simply return a string:

    ```ruby
get('/') do
      slim :index
    end
    

This renders `views/index.slim`.

Instead of a template name, you can also just pass in the template content directly:

```ruby
get('/') do
  code = "= Time.now"
  slim code
end
```    

Templates take a second argument, the options hash:

```ruby
get('/') do
  slim :index, :layout => :post
end
```  

This will render `views/index.slim` embedded in the `views/post.slim` (default is `views/layout.slim`, if it exists).

Options passed to the render method override options set via `set`.

Available Options:

* locals

List of locals passed to the document. Handy with partials. Example: erb "<%= foo %>", :locals => {:foo => "bar"}

* default\_encoding

String encoding to use if uncertain. Defaults to settings.default\_encoding.

* views

Views folder to load templates from. Defaults to settings.views.

* layout

Whether to use a layout (true or false). If it's a Symbol, specifies what template to use. Example: erb :index, :layout => !request.xhr?

* content\_type

Content-Type the template produces. Default depends on template language.

* scope

Scope to render template under. Defaults to the application instance. If you change this, instance variables and helper methods will not be available.

* layout\_engine

Template engine to use for rendering the layout. Useful for languages that do not support layouts otherwise. Defaults to the engine used for the template. Example: set :rdoc, :layout\_engine => :erb

* layout\_options

Special options only used for rendering the layout. Example: set :rdoc, :layout\_options => { :views => 'views/layouts' }

Templates are assumed to be located directly under the `./views` directory. To use a different views directory:

```ruby
set :views, settings.root + '/templates'
``` 

One important thing to remember is that you always have to reference templates with symbols, even if they’re in a subdirectory (in this case, use: `:'subdir/template'` or `'subdir/template'.to_sym`). You must use a symbol because otherwise rendering methods will render any strings passed to them directly.

### Accessing Variables in Templates

Templates are evaluated within the same context as route handlers. Instance variables set in route handlers are directly accessible by templates:

Specify an explicit Hash of local variables:

```ruby
get('/:id') do
  foo = params['id']
  slim :index, :locals => { :bar => foo }
end
```

This is typically used when rendering templates as partials from within other templates.

Filters
-------

Before filters are evaluated before each request within the same context as the routes will be and can modify the request and response. Instance variables set in filters are accessible by routes and templates:

```ruby
before do
  @note = 'Hi!'
  request.path_info = '/foo/bar/baz'
end
```
    
```ruby
get('/foo/*') do
  @note #=> 'Hi!'
  params['splat'] #=> 'bar/baz'
end
```    

After filters are evaluated after each request within the same context as the routes will be and can also modify the request and response. Instance variables set in before filters and routes are accessible by after filters:

```ruby
after do
  puts response.status
end
```

Note: Unless you use the `body` method rather than just returning a String from the routes, the body will not yet be available in the after filter, since it is generated later on.

Filters optionally take a pattern, causing them to be evaluated only if the request path matches that pattern:

```ruby
before('/protected/*') do
  authenticate!
end
```

```ruby
after('/create/:slug') do |slug|
  session[:last_slug] = slug
end
```

Helpers
-------

Use the top-level `helpers` method to define helper methods for use in route handlers and templates:

```ruby
helpers do
  def bar(name)
    "#{name}bar"
  end
end
```
    
```ruby
get('/:name') do
  bar(params['name'])
end
```

Alternatively, helper methods can be separately defined in a module:

```ruby
module FooUtils
  def foo(name) "#{name}foo" end
end

module BarUtils
  def bar(name) "#{name}bar" end
end

helpers FooUtils, BarUtils
```
    

The effect is the same as including the modules in the application class.

### Using Sessions

A session is used to keep state during requests. If activated, you have one session hash per user session:

    enable :sessions
    
```ruby
get('/') do
  "value = " << session[:value].inspect
end
```
    
```ruby
get('/:value') do
  session['value'] = params['value']
end
```

### Halting

To immediately stop a request within a filter or route use:

```ruby
halt
```
    

You can also specify the status when halting:

```ruby
halt 410
```
    

Or the body:

```ruby
halt 'this will be the body'
```
    

Or both:

```ruby
halt 401, 'go away!'
```
    

With headers:

```ruby
halt 402, {'Content-Type' => 'text/plain'}, 'revenge'
```
    

It is of course possible to combine a template with `halt`:

```ruby
halt slim(:error)
```
    

### Passing

A route can punt processing to the next matching route using `pass`:

```ruby
get('/guess/:who') do
  pass unless params['who'] == 'Frank'
  'You got me!'
end
```

```ruby
get('/guess/*') do
  'You missed!'
end
```
    

The route block is immediately exited and control continues with the next matching route. If no matching route is found, a 404 is returned.

### Setting Body, Status Code and Headers

It is possible and recommended to set the status code and response body with the return value of the route block. However, in some scenarios you might want to set the body at an arbitrary point in the execution flow. You can do so with the `body` helper method. If you do so, you can use that method from there on to access the body:

```ruby
get('/foo') do
  body "bar"
end
```

```ruby 
after do
  puts body
end
``` 

It is also possible to pass a block to `body`, which will be executed by the Rack handler (this can be used to implement streaming, see “Return Values”).

Similar to the body, you can also set the status code and headers:

```ruby
get('/foo') do
  status 418
  headers \
    "Allow"   => "BREW, POST, GET, PROPFIND, WHEN",
    "Refresh" => "Refresh: 20; http://www.ietf.org/rfc/rfc2324.txt"
  body "I'm a tea pot!"
end
``` 

Like `body`, `headers` and `status` with no arguments can be used to access their current values.

### Logging

In the request scope, the `logger` helper exposes a `Logger` instance:

```ruby
get('/') do
  logger.info "loading data"
  # ...
end
```
    

This logger will automatically take your Rack handler’s logging settings into account. If logging is disabled, this method will return a dummy object, so you do not have to worry about it in your routes and filters.

### Browser Redirect

You can trigger a browser redirect with the `redirect` helper method:

```ruby
get('/foo') do
  redirect to('/bar')
end
```
    

Any additional parameters are handled like arguments passed to `halt`:

```ruby
redirect to('/bar'), 303
redirect 'http://www.google.com/', 'wrong place, buddy'
```
    

You can also easily redirect back to the page the user came from with `redirect back`:

```ruby
get('/foo') do
  "<a href='/bar'>do something</a>"
end

get('/bar') do
  do_something
  redirect back
end
```
    

To pass arguments with a redirect, either add them to the query:

```ruby
redirect to('/bar?sum=42')
```
    

Or use a session:

```ruby
enable :sessions


get('/foo') do
  session[:secret] = 'foo'
  redirect to('/bar')
end


get('/bar') do
  session[:secret]
end
```
    

### Sending Files

To return the contents of a file as the response, you can use the `send_file` helper method:

```ruby
get('/') do
  send_file 'foo.png'
end
```
    

It also takes options:

```ruby
send_file 'foo.png', :type => :jpg
```
    

The options are:

filename

File name to be used in the response, defaults to the real file name.

last\_modified

Value for Last-Modified header, defaults to the file's mtime.

type

Value for Content-Type header, guessed from the file extension if missing.

disposition

Value for Content-Disposition header, possible values: nil (default), :attachment and :inline

length

Value for Content-Length header, defaults to file size.

status

Status code to be sent. Useful when sending a static file as an error page. If supported by the Rack handler, other means than streaming from the Ruby process will be used. If you use this helper method, Sinatra will automatically handle range requests.

### Accessing the Request Object

The incoming request object can be accessed from request level (filter, routes, error handlers) through the `request` method:

```ruby
# app running on http://example.com/example
get('/foo') do
  t = %w[text/css text/html application/javascript]
  request.accept              # ['text/html', '*/*']
  request.accept? 'text/xml'  # true
  request.preferred_type(t)   # 'text/html'
  request.body                # request body sent by the client (see below)
  request.scheme              # "http"
  request.script_name         # "/example"
  request.path_info           # "/foo"
  request.port                # 80
  request.request_method      # "GET"
  request.query_string        # ""
  request.content_length      # length of request.body
  request.media_type          # media type of request.body
  request.host                # "example.com"
  request.get?                # true (similar methods for other verbs)
  request.form_data?          # false
  request["some_param"]       # value of some_param parameter. [] is a shortcut to the params hash.
  request.referrer            # the referrer of the client or '/'
  request.user_agent          # user agent (used by :agent condition)
  request.cookies             # hash of browser cookies
  request.xhr?                # is this an ajax request?
  request.url                 # "http://example.com/example/foo"
  request.path                # "/example/foo"
  request.ip                  # client IP address
  request.secure?             # false (would be true over ssl)
  request.forwarded?          # true (if running behind a reverse proxy)
  request.env                 # raw env hash handed in by Rack
end
```
    
### Attachments

You can use the `attachment` helper to tell the browser the response should be stored on disk rather than displayed in the browser:

```ruby
get('/') do
  attachment
  "store it!"
end
```
    

You can also pass it a file name:

```ruby
get('/') do
  attachment "info.txt"
  "store it!"
end
```
    

### Dealing with Date and Time

Sinatra offers a `time_for` helper method that generates a Time object from the given value. It is also able to convert `DateTime`, `Date` and similar classes:

```ruby
get('/') do
  pass if Time.now > time_for('Dec 23, 2016')
  "still time"
end
```
    

This method is used internally by `expires`, `last_modified` and akin. You can therefore easily extend the behavior of those methods by overriding `time_for` in your application:

```ruby
helpers do
  def time_for(value)
    case value
    when :yesterday then Time.now - 24*60*60
    when :tomorrow  then Time.now + 24*60*60
    else super
    end
  end
end

get('/') do
  last_modified :yesterday
  expires :tomorrow
  "hello"
end
```

Environments
------------

There are three predefined `environments`: `"development"`, `"production"` and `"test"`. Environments can be set through the `APP_ENV` environment variable. The default value is `"development"`. In the `"development"` environment all templates are reloaded between requests, and special `not_found` and `error` handlers display stack traces in your browser. In the `"production"` and `"test"` environments, templates are cached by default.

To run different environments, set the `APP_ENV` environment variable:

    APP_ENV=production ruby my_app.rb
    

You can use predefined methods: `development?`, `test?` and `production?` to check the current environment setting:

```ruby
get('/') do
  if settings.development?
    "development!"
  else
    "not development!"
  end
end
```

Error Handling
--------------

Error handlers run within the same context as routes and before filters, which means you get all the goodies it has to offer, like `slim`, `erb`, `halt`, etc.

### Not Found

When a `Sinatra::NotFound` exception is raised, or the response’s status code is 404, the `not_found` handler is invoked:
```ruby
not_found do
  'This is nowhere to be found.'
end
```
    

### Error

The `error` handler is invoked any time an exception is raised from a route block or a filter. But note in development it will only run if you set the show exceptions option to `:after_handler`:

```ruby
set :show_exceptions, :after_handler
```
    

The exception object can be obtained from the `sinatra.error` Rack variable:

```ruby
error do
  'Sorry there was a nasty error - ' + env['sinatra.error'].message
end
```
    

Custom errors:

```ruby
error MyCustomError do
  'So what happened was...' + env['sinatra.error'].message
end
```
    

Then, if this happens:

```ruby
get('/') do
  raise MyCustomError, 'something bad'
end
``` 

You get this:

    So what happened was... something bad
    

Alternatively, you can install an error handler for a status code:

```ruby
error 403 do
  'Access forbidden'
end

get('/secret') do
  403
end
```
    

Or a range:

```ruby
error 400..510 do
  'Boom'
end
```
    

Sinatra installs special `not_found` and `error` handlers when running under the development environment to display nice stack traces and additional debugging information in your browser.

### Using a Classic Style Application with a config.ru

Write your app file:

```ruby
# app.rb
require 'sinatra'

get('/') do
  'Hello world!'
end
```
    
x§
And a corresponding `config.ru`:

```ruby
require './app'
run Sinatra::Application
```
    

### When to use a config.ru?

A `config.ru` file is recommended if:

*   You want to deploy with a different Rack handler (Passenger, Unicorn, Heroku, …).
*   You want to use more than one subclass of `Sinatra::Base`.
*   You want to use Sinatra only for middleware, and not as an endpoint.

**There is no need to switch to a `config.ru` simply because you switched to the modular style, and you don’t have to use the modular style for running with a `config.ru`.**

Scopes and Binding
------------------

The scope you are currently in determines what methods and variables are available.

### Request/Instance Scope

For every incoming request, a new instance of your application class is created, and all handler blocks run in that scope. From within this scope you can access the `request` and `session` objects or call rendering methods like `erb` or `slim`. You can access the application scope from within the request scope via the `settings` helper:

```ruby
# Hey, I'm in the application scope!
get('/define_route/:name') do
  # Request scope for '/define_route/:name'
  @value = 42

  settings.get("/#{params['name']}") do
    # Request scope for "/#{params['name']}"
    @value # => nil (not the same request)
  end

  "Route defined!"
end
```
    

You have the request scope binding inside:

*   get, head, post, put, delete, options, patch, link and unlink blocks
*   before and after filters
*   helper methods
*   templates/views

### Delegation Scope

The delegation scope just forwards methods to the class scope. However, it does not behave exactly like the class scope, as you do not have the class binding. Only methods explicitly marked for delegation are available, and you do not share variables/state with the class scope (read: you have a different `self`). You can explicitly add method delegations by calling `Sinatra::Delegator.delegate :method_name`.

You have the delegate scope binding inside:

*   The top level binding, if you did `require "sinatra"`
*   An object extended with the `Sinatra::Delegator` mixin

Have a look at the code for yourself: here’s the [Sinatra::Delegator mixin](https://github.com/sinatra/sinatra/blob/ca06364/lib/sinatra/base.rb#L1609-1633) being [extending the main object](https://github.com/sinatra/sinatra/blob/ca06364/lib/sinatra/main.rb#L28-30).

Command Line
------------

Sinatra applications can be run directly:

    ruby myapp.rb [-h] [-x] [-q] [-e ENVIRONMENT] [-p PORT] [-o HOST] [-s HANDLER]
    

Options are:

    -h # help
    -p # set the port (default is 4567)
    -o # set the host (default is 0.0.0.0)
    -e # set the environment (default is development)
    -s # specify rack server/handler (default is thin)
    -q # turn on quiet mode for server (default is off)
    -x # turn on the mutex lock (default is off)
    

Further Reading
---------------

*   [Project Website](http://www.sinatrarb.com/) - Additional documentation, news, and links to other resources.
*   [Contributing](http://www.sinatrarb.com/contributing) - Find a bug? Need help? Have a patch?
*   [Issue tracker](https://github.com/sinatra/sinatra/issues)
*   [Twitter](https://twitter.com/sinatra)
*   [Mailing List](http://groups.google.com/group/sinatrarb/topics)
*   IRC: [#sinatra](irc://chat.freenode.net/#sinatra) on http://freenode.net
*   [Sinatra & Friends](https://sinatrarb.slack.com/) on Slack and see [here](https://sinatra-slack.herokuapp.com/) for an invite.
*   [Sinatra Book](https://github.com/sinatra/sinatra-book/) Cookbook Tutorial
*   [Sinatra Recipes](http://recipes.sinatrarb.com/) Community contributed recipes
*   API documentation for the [latest release](http://www.rubydoc.info/gems/sinatra) or the [current HEAD](http://www.rubydoc.info/github/sinatra/sinatra) on http://www.rubydoc.info/
*   [CI server](https://travis-ci.org/sinatra/sinatra)