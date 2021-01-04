# Understanding Rbenv, RubyGems & Bundler

Managing dependencies in Ruby usually involves specifying Ruby versions and gem versions that our project relies upon. In my experience working in Ruby, debugging dependencies has been my biggest pain point. Although failures aren't common because "they just work", but when things go wrong, they're usually hard to debug and fix. In this article, I'm going to lay out the pieces involved in dependency management in Ruby. This will assist in debugging these strange issues when they occur.

![cover](cover.png)

### Ruby Code Loading

The Ruby language by default provides 2 major methods for loading code defined elsewhere `load` & `require`. 

```ruby
load 'json.rb'
require 'json.rb'
require_relative 'json.rb'
```

Both methods of loading accept both absolute & relative paths as arguments. However, there a number differentiating factors

1. Multiple calls to `load` will re-execute the file, whereas multiple calls to `require` will not re-execute the file rather will return `false`.
2. Calls to `load` resolves only to absolute & relative paths. Calls to `require` checks up on the `$LOAD_PATH` when the path doesn't resolve to an absolute path.

A third variant is `require_relative` which uses relative paths to require code relative to the current files location not the ruby's process working directory.

### Rbenv

Rbenv is a ruby version manager used to manage different versions of Ruby. There's also [Ruby Version Manager (RVM)](https://rvm.io) that tries to solve the same problem, In this article we'd be focusing on Rbenv. Using Rbenv there are two concepts to understand - Shims & Rehashing. 

- **Shims:** These ****are lightweight bash scripts that exist in your `PATH` and intercepts commands. They determine the appropriate version to run the respective command and pass the command and it's arguments to that version. Rbenv shims can be found at `~/rbenv/shims`. They must as well exist on your `PATH`.

    **Caveat:** Shims on your `PATH` must be prepended, this ensures they're the first point of contact for your ruby executables and they can properly intercept. The best way I found to understand your `PATH` setup and know if your shims are intercepting properly or not is 

    ```ruby
    # bash
    > which -a bundle

    # Output
    /Users/subomioluwalana/.rbenv/shims/bundle
    /usr/bin/bundle
    ```

    `which -a bundle` because this will naively look through your PATH and print out in order where `bundle` can be found. If anything is printed before anything in `~/.rbenv/shims`. It means your shims aren’t setup properly. `rbenv which bundle` won't reveal this because that command works in context of `rbenv` not searching your `PATH`.

- **Rehashing:** This is the process of creating shims, when you newly install a ruby gem that provides an executable like `rspec` , you need to run `rbenv rehash` to create the shim, so subsequent calls to `rspec` can be intercepted by `rbenv` and passed on to the appropriate ruby version.

### RubyGems

Next is RubyGems. From the Official ruby site; RubyGems is a Ruby packaging system designed to facilitate the creation, sharing and installation of libraries (in some ways, it is a distribution packaging system similar to, say, apt-get, but targeted at Ruby software). 

A gem is a bunch of related code to solve a specific problem. To install a gem & get information about the gem environment: 

```ruby
gem install gemname
```

```ruby
gem env

# Output
RubyGems Environment:  
	- RUBYGEMS VERSION: 3.1.2  
	- RUBY VERSION: 2.7.1 (2020-03-31 patchlevel 83) [x86_64-darwin20]  
	- INSTALLATION DIRECTORY: /Users/subomioluwalana/.rbenv/versions/2.7.1/lib/ruby/gems/2.7.0  
	- USER INSTALLATION DIRECTORY: /Users/subomioluwalana/.gem/ruby/2.7.0  
	- RUBY EXECUTABLE: /Users/subomioluwalana/.rbenv/versions/2.7.1/bin/ruby  
	- GIT EXECUTABLE: /usr/bin/git  
	- EXECUTABLE DIRECTORY: /Users/subomioluwalana/.rbenv/versions/2.7.1/bin  
	- SPEC CACHE DIRECTORY: /Users/subomioluwalana/.gem/specs  
	- SYSTEM CONFIGURATION DIRECTORY: /Users/subomioluwalana/.rbenv/versions/2.7.1/etc  
	- RUBYGEMS PLATFORMS:    
		- ruby    
		- x86_64-darwin-20  
	- GEM PATHS:     
		- /Users/subomioluwalana/.rbenv/versions/2.7.1/lib/ruby/gems/2.7.0     
		- /Users/subomioluwalana/.gem/ruby/2.7.0  
	- GEM CONFIGURATION:     
		...
	- REMOTE SOURCES:     
		- https://rubygems.org/  
	- SHELL PATH:     
		- /Users/subomioluwalana/.rbenv/versions/2.7.1/bin
```

Gems are usually installed at  `~/.rbenv/versions/{version-number}/lib/ruby/gems/{minor-version}/`. Although this works "natively" it is because Ruby comes with RubyGems by default since version 1.9, previous Ruby versions require RubyGems to be installed by hand. How does RubyGems actually solve this problem? It monkey patches the `Kernel`'s require system with its own `require` method. With this in-place, when `require honeybadger` is called it searches through the gems folder for `honeybadger.rb` when it finds the gem, it activates it.

- E.g. `require 'honeybadger'` produces something similar to:
    - `spec = Gem::Specification.find_by_path('honeybadger')`
    - `spec.activate`

Activating a gem simply means, putting it up in the `$LOAD_PATH`. RubyGems also helps download all gems dependencies before downloading the gem itself.  

### Bundler

Bundler helps us easily specify all our dependencies and optionally a specific version. It then resolves our gems installs it & it's own dependencies. 

- Bundler resolves dependencies and generates a lockfile

    ```ruby
    # Gemfile
    gem 'httparty'
    ```

    If we run `bundle` or `bundle install` this will generate the lockfile below: 

    ```ruby
    GEM
      specs:
        httparty (0.18.1)
          mime-types (~> 3.0)
          multi_xml (>= 0.5.2)
        mime-types (3.3.1)
          mime-types-data (~> 3.2015)
        mime-types-data (3.2020.1104)
        multi_xml (0.6.0)

    PLATFORMS
      ruby

    DEPENDENCIES
      httparty

    BUNDLED WITH
       2.1.4
    ```

    It resolves the dependencies for `httparty` by finding the suitable version for it's dependencies and specifying those. As well, bundler also tries to resolve dependencies between gems. E.g.

    ```ruby
    # Gemfile
    gem 'httparty' # That relies on gem 'mime-types', '>= 3.0.1, < 4.0.1'
    gem 'rest-client' # That relies on gem 'mime-types', '>= 2.0.1, < 3.0'
    ```

    The example above is arbitrary and will result in an error like: 

    ```ruby
    Bundler could not find compatible versions for gem "mime-types":
    In Gemfile:
    	httparty was resolved to 0.18.1, which depends on 
    		mime-types ('>= 3.0.1, < 4.0.1')

    	rest-client was resolved to 2.0.4, which depends on 
    		mime-types ('>= 2.0.1, < 3.0')
    ```

    This is because two gems have dependencies that are not compatible and cannot be automatically resolved. 

    The lock file ensures that our project dependencies are consistent across environments (development, staging or production). And by best practices should be checked into version control. 

- Bundler doesn't allow me see what I didn't specify. In a sample gemfile like:

    ```ruby
    # Gemfile
    gem 'httparty'

    # irb
    require 'rest-client'

    # raises
    LoadError (cannot load such file -- rest-client)
    ```

    This ensures that only the dependencies specified in our `Gemfile` can be required by our project.

- Bundle exec

    When you run `rspec` in a project directory you run into the possible problem of running a different version than what was specified in the `Gemfile`. This is because the most recent version will be selected to run vs. the version specified in the `Gemfile`. `bundle exec rspec` ensures `rspec` is run in the context of that project i.e. The gems  specified in the Gemfile.

- Bundle binstubs

    Often times, we read articles where we run commands like `./bin/rails` this command is similar to `bundle exec rails`. Binstubs are wrappers around ruby executables, to easy the usage of `bundle exec`. 

    - To generate a binstub for a gem type `bundle binstubs gem-name`. This creates a binstub in the `./bin` folder but can be configured with the `--path` directory if set.

### References

[How Do Gems Work?](https://www.justinweiss.com/articles/how-do-gems-work/)

[Rbenv](https://github.com/rbenv/rbenv)

[RubyGems](https://rubygems.org)

[Bundler](https://bundler.io)
