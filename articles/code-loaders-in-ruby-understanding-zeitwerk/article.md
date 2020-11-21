# Code Loaders in Ruby - Understanding Zeitwerk

> With Zeitwerk you can streamline your programming knowing that classes and modules are available everywhere.

## What are Code Loaders?

Code Loaders let developers define `classes` and `modules` across different files and folders and use them throughout the codebase without explicitly requiring them. Rails is a good example of a piece of software that uses code loaders. Programming in Rails, doesn't require  explicit `require` calls to load the models before using it in the controllers. In fact, in Rails 6, everything in the `app` directory is auto-loaded on app boot with a few exceptions. 

While it is easy to think Code loading is all about making calls to `require`, it isn't that simple. Code loading can be further broken down into 3 parts

- Auto Loading: This means code is loaded on-the-fly as it is required. As an example in Rails, running `rails s` doesn't load all the models, controllers etc. Just yet. But on the first hit of the model `User` it runs the auto loading mechanism to find and use the model. This is auto loading in action. This has some advantages for our development environment, we have faster app and `rails console` startup times. `Rails.config.autoload_path` controls the paths to be auto-loaded.
- Eager Loading: This means code is loaded into memory at app startup and doesn't wait for the constants to be called before requiring it. In Rails, code is eager loaded in production. From the explanation above, the autoloading code in production will result in slow response times as each constant will be required on-the-fly. `Rails.config.eager_load_paths` controls the paths to be eager loaded.
- Reloading: Here,  the code loader will constantly be watching for changes on all files in the `autoload_path` and reload any file it notices any changes. In Rails, this can be quite useful in development as it enables us to run `rails s` and simultaneously make changes without the need to restart the rails server. This is reloading in action.

Easily, we can see most of these concepts have been developed & live in Rails. Zeitwerk changes this! Zeitwerk enables us to bring all the code loading action to any Ruby project.

## What is Zeitwerk?

Zeitwerk is an efficient and thread-safe code loader for Ruby which can be used in any Ruby project; Web frameworks (Rails, Hanami, Sinatra) Cli tools, gems, etc. With it you can streamline your programming knowing that classes and modules are available everywhere. Traditionally, Rails and *some* other gems have inbuilt code loaders to enable this functionality. However, Zeitwerk extracts these concepts into a gem and allows any Rubyist to apply these concepts to their projects. 

### Installing Zeitwerk

First things first, we need to install the gem:

```ruby
gem install zeitwerk

# OR in your Gemfile
gem 'zeitwerk', '~> 2.4.0'
```

### Configuring Zeitwerk

So let's start with the basics:

```ruby
require 'zeitwerk'
loader = Zeitwerk::Loader.new
...
loader.setup
```

The above code instantiates a loader instance and calls `setup`.  After the call to `setup`, loaders are ready to load code. But before that, all necessary configurations on the `loader` should be covered already. In this article, I'd cover a few of the configurations on the `loader` and conventions on how to structure your code.

- File Structure: For Zeitwerk to work, files and directories names need to match the modules and class names they define. E.g.

    ```ruby
    lib/my_gem.rb         -> MyGem
    lib/my_gem/foo.rb     -> MyGem::Foo
    lib/my_gem/bar_baz.rb -> MyGem::BarBaz
    lib/my_gem/woo/zoo.rb -> MyGem::Woo::Zoo
    ```

- Root Namespaces: Root Namespaces are directories where `Zeitwerk` can find your code.  When `modules` and `classes` are referenced, Zeitwerk knows to search the root namespaces with matching file name. E.g.

    ```ruby
    require 'zeitwerk'
    loader = Zeitwerk::Loader.new
    loader.push_dir("app/models")
    loader.push_dir("app/controllers")

    // matches as follows
    app/models/user.rb                        -> User
    app/controllers/admin/users_controller.rb -> Admin::UsersController
    ```

    There are two primary ways of defining the root namespace for two different use cases. The default way is as shown below: 

    ```ruby
    // init.rb 
    require 'zeitwerk'
    loader = Zeitwerk::Loader.new
    loader.push_dir("#{__dir__}/bar")
    ...
    loader.setup

    // bar/foo.rb
    class Foo; end
    ```

    This means the class `Foo` can be referenced without an explicit `Bar::Foo` as the bar directory acts as a root namespace. The second way to define a namespace is to explicitly state the namespace in the call to `push_dir`

    ```ruby
    // init.rb
    require 'zeitwerk'

    module Bar
    end
    loader = Zeitwerk::Loader.new
    loader.push_dir("#{__dir__}"/bar, namespace: Bar)
    loader.setup

    // src/foo.rb
    class Bar::Foo; end
    ```

    There are a few things to note from this code:

    1. The module `Bar` was already defined before being used by `push_dir` . If the module we want to use is defined by a third-party, then a simple require will define it before we use it in our call to `push_dir`.
    2. The `push_dir` explicitly specifies the namespace `Bar`
    3. The file `src/foo.rb` defined `Bar::Foo` not `Foo` and did not need to create the directory like `src/bar/foo.rb`.
- Independent code loader: By design, Zeitwerk allows each project or app dependency to manage their individual project tree. This means the code loading mechanism of each dependency is managed by that dependency. As an example, in Rails 6, Zeitwerk handles code loading for the rails app, and allows each gem dependency to manage their own project tree separately. It is an error condition to have overlapping files between multiple code loaders.
- Autoloading: With the above setup, once the call to `setup` is made. All classes & modules will be available on demand.
- Reloading: To enable reloading, the `loader` has to be explicity configured for it. E.g.

    ```ruby
    loader = Zeitwerk::Loader.new
    ...
    loader.enable_reloading # you need to opt-in before setup
    loader.setup
    ...
    loader.reload
    ```

    `loader.reload` call reloads the project tree on-the-fly and any new changes are visible immediately. However, we still need a surrounding mechanism to detect changes on the file system and call `loader.reload`. A simple version is shown below: 

    ```ruby
    require 'filewatcher'

    loader = Zeitwerk::Loader.new
    ...
    loader.enable_reloading
    loader.setup
    ...

    my_filewatcher = Filewatcher.new('lib/')
    Thread.new(my_filewatcher) {|fw| fw.watch {|filename| loader.reload } }
    ```

### Using Zeitwerk in Rails

Zeitwerk is enabled by default in Rails 6.0. However, you can opt-out of it and use the Rails `classic` code loader. 

```ruby
# config/application.rb
config.load_defaults "6.0"
config.autoloader = :classic
```

### Using Zeitwerk in gems

Zeitwerk provides a convenient method for gems as long as they are using the standard gem structure (`lib/special_gem`). This convenience method can be used as follows:

```ruby
// lib/special_gem.rb
require 'zeitwerk'

module SpecialGem
end

loader = Zeitwerk::Loader.for_gem
loader.setup
```

With the standard gem structure, the `for_gem` call adds the `lib` directory as a root namespace, enabling every code in the `lib` directory to be found automatically.

For more inspiration, you can checkout gems using Zeitwerk:

- [Karafka](https://github.com/karafka/karafka)
- [Jets](https://github.com/boltops-tools/jets)

## References

[Rails autoloading — how it works, and when it doesn't](https://www.urbanautomaton.com/blog/2013/08/27/rails-autoloading-hell/)

[Zeitwerk](https://github.com/fxn/zeitwerk)