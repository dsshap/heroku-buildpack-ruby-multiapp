Heroku buildpack: Multiple apps in one repo
======================

This buildpack allows you to have two applications in one repo, but only having one application compiled and run on the heroku server at a time. 

This is a [Heroku buildpack](http://devcenter.heroku.com/articles/buildpacks) for Ruby Rails3 apps. It uses [Bundler](http://gembundler.com) for dependency management.


Usage
-----

### Config Vars

Example Usage:

    === multiapp-app1 Config Vars
      BUILDPACK_URL:  https://github.com/dsshap/heroku-buildpack-ruby-multiapp.git
      BUILD_DIR:      app1
      BUNDLE_GEMFILE: /app/app1/Gemfile               *Dont set this until after the slug has been compiled*
      GEM_PATH:       app1/vendor/bundle/ruby/1.9.1
      LANG:           en_US.UTF-8
      PATH:           bin:app1/vendor/bundle/ruby/1.9.1/bin:/usr/local/bin:/usr/bin:/bin
      RACK_ENV:       production
      RAILS_ENV:      production

Procfile
    
    web: bundle exec thin start -R $BUILD_DIR/config.ru -p $PORT -e $RACK_ENV 

### Ruby

Example Usage:

    $ ls
    Gemfile Gemfile.lock

    $ heroku create --stack cedar --buildpack https://github.com/heroku/heroku-buildpack-ruby.git

    $ git push heroku master
    ...
    -----> Heroku receiving push
    -----> Fetching custom git buildpack... done
    -----> multi app detected
           @build_path: /tmp/build_2vvd5w39yvspj/app1
    -----> Installing dependencies using Bundler version 1.2.1
           Running: bundle install --without development:test --path vendor/bundle --binstubs bin/ --deployment
           Installing rake (0.9.2.2)
           Using bundler (1.2.1)
           Your bundle is complete! It was installed into ./vendor/bundle
           Cleaning up the bundler cache.
    -----> Discovering process types
           Procfile declares types -> web
           Default types for multi -> console, rake



The buildpack will detect your app as Ruby if it has a `Gemfile` and `Gemfile.lock` files in the root directory. It will then proceed to run `bundle install` after setting up the appropriate environment for [ruby](http://ruby-lang.org) and [Bundler](http://gembundler.com).

#### Bundler

For non-windows `Gemfile.lock` files, the `--deployment` flag will be used. In the case of windows, the Gemfile.lock will be deleted and Bundler will do a full resolve so native gems are handled properly. The `vendor/bundle` directory is cached between builds to allow for faster `bundle install` times. `bundle clean` is used to ensure no stale gems are stored between builds.

### Rails 2

Example Usage:

    $ ls
    app  config  db  doc  Gemfile  Gemfile.lock  lib  log  public  Rakefile  README  script  test  tmp  vendor

    $ ls config/environment.rb
    config/environment.rb

    $ heroku create --stack cedar --buildpack https://github.com/heroku/heroku-buildpack-ruby.git

    $ git push heroku master
    ...
    -----> Heroku receiving push
    -----> Ruby/Rails app detected
    -----> Installing dependencies using Bundler version 1.1.rc
    ...
    -----> Writing config/database.yml to read from DATABASE_URL
    -----> Rails plugin injection
           Injecting rails_log_stdout
    -----> Discovering process types
           Procfile declares types      -> (none)
           Default types for Ruby/Rails -> console, rake, web, worker

The buildpack will detect your app as a Rails 2 app if it has a `environment.rb` file in the `config`  directory.

#### Rails Log STDOUT
  A [rails_log_stdout](http://github.com/ddollar/rails_log_stdout) is installed by default so Rails' logger will log to STDOUT and picked up by Heroku's [logplex](http://github.com/heroku/logplex).

#### Auto Injecting Plugins

Any vendored plugin can be stopped from being installed by creating the directory it's installed to in the slug. For instance, to prevent rails_log_stdout plugin from being injected, add `vendor/plugins/rails_log_stdout/.gitkeep` to your git repo.

### Rails 3

Example Usage:

    $ ls
    app  config  config.ru  db  doc  Gemfile  Gemfile.lock  lib  log  Procfile  public  Rakefile  README  script  tmp  vendor

    $ ls config/application.rb
    config/application.rb

    $ heroku create --stack cedar --buildpack https://github.com/heroku/heroku-buildpack-ruby.git

    $ git push heroku master
    -----> Heroku receiving push
    -----> Ruby/Rails app detected
    -----> Installing dependencies using Bundler version 1.1.rc
           Running: bundle install --without development:test --path vendor/bundle --deployment
           ...
    -----> Writing config/database.yml to read from DATABASE_URL
    -----> Preparing app for Rails asset pipeline
           Running: rake assets:precompile
    -----> Rails plugin injection
           Injecting rails_log_stdout
           Injecting rails3_serve_static_assets
    -----> Discovering process types
           Procfile declares types      -> web
           Default types for Ruby/Rails -> console, rake, worker

The buildpack will detect your apps as a Rails 3 app if it has an `application.rb` file in the `config` directory.

#### Assets

To enable static assets being served on the dyno, [rails3_serve_static_assets](http://github.com/pedro/rails3_serve_static_assets) is installed by default. If the [execjs gem](http://github.com/sstephenson/execjs) is detected then [node.js](http://github.com/joyent/node) will be vendored. The `assets:precompile` rake task will get run if no `public/manifest.yml` is detected.  See [this article](http://devcenter.heroku.com/articles/rails31_heroku_cedar) on how rails 3.1 works on cedar.

Hacking
-------

To use this buildpack, fork it on Github.  Push up changes to your fork, then create a test app with `--buildpack <your-github-url>` and push to it.

To change the vendored binaries for Bundler, [Node.js](http://github.com/joyent/node), and rails plugins, use the rake tasks provided by the `Rakefile`. You'll need an S3-enabled AWS account and a bucket to store your binaries in as well as the [vulcan](http://github.com/ddollar/vulcan) gem to build the binaries on heroku.

For example, you can change the vendored version of Bundler to 1.1.rc.

First you'll need to build a Heroku-compatible version of Node.js:

    $ export AWS_ID=xxx AWS_SECRET=yyy S3_BUCKET=zzz
    $ s3 create $S3_BUCKET
    $ rake gem:install[bundler,1.1.rc]

Open `lib/language_pack/ruby.rb` in your editor, and change the following line:

    BUNDLER_VERSION = "1.1.rc"

Open `lib/language_pack/base.rb` in your editor, and change the following line:

    VENDOR_URL = "https://s3.amazonaws.com/zzz"

Commit and push the changes to your buildpack to your Github fork, then push your sample app to Heroku to test.  You should see:

    -----> Installing dependencies using Bundler version 1.1.rc

NOTE: You'll need to vendor the plugins, node, Bundler, and libyaml by running the rake tasks for the buildpack to work properly.

Flow
----

Here's the basic flow of how the buildpack works:

Ruby (Gemfile and Gemfile.lock is detected)

* runs Bundler
* installs binaries
  * installs node if the gem execjs is detected
* runs `rake assets:precompile` if the rake task is detected

Rack (config.ru is detected)

* everything from Ruby
* sets RACK_ENV=production

Rails 2 (config/environment.rb is detected)

* everything from Rack
* sets RAILS_ENV=production
* install rails 2 plugins
  * [rails_log_stdout](http://github.com/ddollar/rails_log_stdout)

Rails 3 (config/application.rb is detected)

* everything from Rails 2
* install rails 3 plugins
  * [rails3_server_static_assets](https://github.com/pedro/rails3_serve_static_assets)

