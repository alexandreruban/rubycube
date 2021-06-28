---
title: Rebuilding FactoryBot in 150 lines of code
description: This description will be used in the meta tags
date: 2021-06-19
minutes_read: 20
tags: [ruby, rails]
---

`FactoryBot` is a gem that enables you to create fake data for your tests. If you work with Ruby on Rails, there are good chances that you worked with the gem on a few projects.

In this article, we will rebuild a `FactoryBot` clone called `TinyFactory` in about 150 lines of code. We will learn amazing Ruby features such as:

- How the elegant `FactoryBot` syntax works under the hood
- How to create a gem
- How to use blocks and procs
- How to use `#method_missing` and `#instance_eval`

By the end of the article, you will know enough to go through the source code of the real [factory bot repository](https://github.com/thoughtbot/factory_bot) on your own if you want to! To get the most out of this article, you should take your time and build the gem with me, it is guided step by step for you to understand everything! Are you ready to learn some cool things in Ruby? Let's get started!

## Creating the TinyFactory gem

Let's start by creating a new gem called `TinyFactory` with the `bundle gem` command:

```bash
bundle gem tiny_factory
cd tiny_factory
```

If you're creating a gem for the first time, I'll quickly describe the main files that were created for you, otherwise, you can skip directly to the next section!

The `tiny_factory.gempspec` file contains the specification of your gem. It lists various information about the gem, the author, and the list of the dependencies.

The `Rakefile` looks like this by default:

```rb
# frozen_string_literal: true

require "bundler/gem_tasks"
require "rake/testtask"

Rake::TestTask.new(:test) do |t|
  t.libs << "test"
  t.libs << "lib"
  t.test_files = FileList["test/**/*_test.rb"]
end

task default: :test
```

This code means that the default rake task will run all the test files of the `test` folder ending with `_test.rb`. To run this default rake task, simply type `bundle exec rake` in your terminal.

The `test` folder contains the tests you'll write to test your gem. The `test_helper.rb` file comes with some boilerplate code to automatically add the files from your `lib` folder in the load path. We will write some tests in this article later!

Finally, the `lib` folder is where you will write your source code!

All the Ruby gems are built with these conventions, so next time you open the source code of a gem, you will find the same folder hierarchy and you won't be lost!

## Testing the TinyFactory gem

### Adding an integration test

To clearly define what we will build in this article, let's first write an integration test! We will consider that we have succeeded in our mission when all the integration tests are green! Let's create a file called `integration_test.rb` in our `test` folder:

```rb
# test/integration_test.rb

require "test_helper"

class IntegrationTest < Minitest::Test
  def setup
    TinyFactory.define :user do
      first_name { "Alexandre" }
      last_name { "Ruban" }
      email { "#{first_name}@hey.com".downcase }
    end
  end

  def test_attributes_for
    attributes = attributes_for(:user)

    assert_kind_of Hash, attributes

    assert_equal "Alexandre",         attributes[:first_name]
    assert_equal "Ruban",             attributes[:last_name]
    assert_equal "alexandre@hey.com", attributes[:email]
  end

  def test_build
    user = build(:user)

    assert_kind_of User, user
    assert user.new_record?

    assert_equal "Alexandre",         user.first_name
    assert_equal "Ruban",             user.last_name
    assert_equal "alexandre@hey.com", user.email
  end

  def test_create
    user = create(:user)

    assert_kind_of User, user
    assert user.persisted?

    assert_equal "Alexandre",         user.first_name
    assert_equal "Ruban",             user.last_name
    assert_equal "alexandre@hey.com", user.email
  end
end
```

As you can see, the syntax in the `#setup` method is close to the real `FactoryBot` syntax. This is what we will build step by step in this article!

To make our test pass later, we still need to add a little bit of boilerplate.

### Adding the required dependencies

To make our test pass, we need to add a `User` model to our application. This `User` model will inherit from `ActiveRecord::Base` to simulate a real model in a Rails application.

Let's add all the dependencies we need in the gemspec:

```rb
# tiny_factory.gemspec

require_relative "lib/tiny_factory/version"

Gem::Specification.new do |spec|
  # ...
  # Don't remove the automatically generated code
  # Add these dependencies at the end

  spec.add_development_dependency("activerecord", "~> 6.1")
  spec.add_development_dependency("sqlite3", "~> 1.4.2")
  spec.add_dependency("activesupport", "~> 6.1")
end
```

As you can see, we add a development dependency on active record and sqlite3. This is because we'll need a `User` model inheriting from `ActiveRecord::Base`. We will also need to save `User` instances in a sqlite3 database in memory.

Note that we also added a dependency on active support that we will talk about later in the article.

Now that we listed the dependencies, we can run the `bundle install` command. Bundler might complain that there are some "TODO" in the gemspec file. Removing them should enable you to run `bundle install` without any troubles!

### Creating our User model

Let's create the `User` model in the `test_helper.rb` file:

```rb
# test/test_helper.rb

require "tiny_factory"
require "minitest/autorun"
require "active_record"

ActiveRecord::Base.establish_connection(
  database: ":memory:",
  adapter: "sqlite3"
)

class CreateSchema < ActiveRecord::Migration[6.0]
  def self.up
    create_table :users, force: true do |t|
      t.string  :first_name
      t.string  :last_name
      t.string  :email
    end
  end
end

CreateSchema.migrate(:up)

class User < ActiveRecord::Base
  validates :first_name, :last_name, :email, presence: true
end
```

This piece of code creates a new connection to a sqlite3 database in memory, runs a migration that creates the `users` table, and creates the `User` model that inherits from `ActiveRecord::Base` just like you would do in a Rails application. As we required the "test_helper" in our `integration_test.rb` file, the `User` model will be available here as well!

Now that our test files and our dependencies are all set, we are ready to dive into the `FactoryBot` code!

## The two steps of FactoryBot

`FactoryBot` works in two steps:

1. You define a factory
2. You run the factory

The definition is what happens when we write those lines in our integration test.

```rb
TinyFactory.define :user do
  first_name { "Alexandre" }
  last_name { "Ruban" }
  email { "#{first_name}@hey.com".downcase }
end
```

We then run the factory when we use one of the methods `#attributes_for`, `#build` or `#create`.

## Defining a Factory

We will start this article by understanding how to define the `:user` factory of our integration test. The definition relies on two simple classes: `Factory` and `Attribute`. Let's understand them!

### The Factory class

The `Factory` class is responsible for holding the factory name and the attributes' definitions! Let's make that clear with a piece of code:

```rb
# lib/tiny_factory/factory.rb

module TinyFactory
  class Factory
    # A factory has a factory_name and holds the attributes
    attr_reader :factory_name

    def initialize(factory_name)
      @factory_name = factory_name
      @attributes = []
    end

    # Here comes the Attribute object!
    def add_attribute(name, definition)
      @attributes << Attribute.new(name, definition)
    end
  end
end
```

In our example, the `factory_name` is `:user` and the attributes are `first_name`, `last_name`, and `email`.

Now that we understand that our `Factory` holds its name and adds attributes  definitions to a list of attributes, let's move on to the `Attribute` class.

### The Attribute class

The `Attribute` class is the simplest of all! It's only responsible for holding an attribute `name` and its `definition`.

```rb
# lib/tiny_factory/attribute.rb

module TinyFactory
  class Attribute
    def initialize(name, definition)
      @name = name.to_sym
      @definition = definition
    end
  end
end
```

Let's take our `email` attribute as an example. When we write `email { "#{first_name}@hey.com".downcase }` in the integration test, what we are creating under the hood is an `Attribute` with its name equals to `:email` and its definition equals to a proc that we could write like this `Proc.new { "#{first_name}@hey.com".downcase }`

Now, if you're not familiar with procs, you will understand them **much more** after reading this article so keep going!

A proc is an object that holds a piece of code that will be evaluated later! Here, the definition `Proc.new { "#{first_name}@hey.com".downcase }` has not been evaluated yet! It is a piece of code waiting to be *called*. We will see later in the article that the definition is evaluated when you run the factory.

### Adding syntactic sugar

Now that we have the `Factory` and the `Attribute` objects, it's time to reveal the mysteries behind this beautiful syntax:

```rb
TinyFactory.define :user do
  first_name { "Alexandre" }
  last_name { "Ruban" }
  email { "#{first_name}@hey.com".downcase }
end
```

With the `Factory` and `Attribute` classes we just coded, we could define a factory like this:

```rb
factory = TinyFactory::Factory.new(:user)
factory.add_attribute(:first_name, Proc.new { "Alexandre" })
factory.add_attribute(:last_name,  Proc.new { "Alexandre" })
factory.add_attribute(:email,      Proc.new { "#{first_name}@hey.com".downcase })
```

This works but it is not very elegant.

We are building a gem that will be used by a lot of developers and we want it to have an easy-to-remember *API*. To make our syntax elegant, we will add *syntactic sugar*. This means that our gem will keep the same features it already has but we will make the *API* easier to write and remember.

Let's transform our current syntax into the elegant syntax. First, we need to add the `TinyFactory.define` method:

```rb
# lib/tiny_factory.rb

# This was generated by the `bundle gem tiny_factory` command
require "tiny_factory/version"

# You need to require the files we just created
require "tiny_factory/factory"
require "tiny_factory/attribute"

module TinyFactory
  def self.define(name, &block)
    factory = Factory.new(name)
    factory.instance_eval(&block)
    factory
  end
end
```

The `Tinyfactory.define` method instanciates a new Factory (with a name of `:user` in our example) and then calls `#instance_eval` with the block on it! What does the `#instance_eval` method do? It evaluates the block in the context of the instance on which it was called. This means that `factory.instance_eval(&block)` is equivalent in our example to writing:

```rb
factory.first_name { "Alexandre" }
factory.last_name  { "Ruban" }
factory.email      { "#{first_name}@hey.com".downcase }
```

All right but our factory does not respond to `#first_name` or `#last_name` or even `#email`! It only responds to `#add_attribute`. Let's change that by adding the `#method_missing` method to our `Factory` class.

What is `#method_missing`? It's a Ruby method that gets called when the method was not found on the object or any of its ancestors! Let's do a small example for you to understand:

```rb
class Random
end

Random.new.email
# => undefined method `email' for #<Random:0x00007fa950a840e8> (NoMethodError)
```

As you can see, calling email on an instance of the `Random` raises a `NoMethodError`. We can change that by adding `method_missing` to the `Random class`

```rb
class Random
  def method_missing(name, *args, &block)
    puts "The method #{name} was called"
    puts "The method was called with #{args} as arguments" if args.any?
    puts "The method was also given a block" if block
  end
end

Random.new.first_name
# => The method first_name was called


Random.new.names("Alexandre", "Ruban")
# => The method names was called
# => The method was called with ["Alexandre", "Ruban"] as arguments

Random.new.email { "#{first_name}@hey.com".downcase }
# => The method email was called
# => The method was also given a block
```

As you can see, `#method_missing` prevents the `NoMethodError`. The name of the missing method that was called on the object becomes the first argument of `#method_missing`. The arguments of the missing method that was called can be retrieved with `*args` which turns those arguments into an array. Last but not least if a block was passed to the missing method it can be transformed as a proc and passed as an argument!

What we want is to make those two syntax equivalent:

```rb
factory.add_attribute(:email) { "#{first_name}@hey.com".downcase }
factory.first_name { "#{first_name}@hey.com".downcase }
```

This can be achieved with `#method_missing` like this:

```rb
# lib/tiny_factory/factory.rb

module TinyFactory
  class Factory
    # All the previous code
    # ...

    def method_missing(name, &block)
      add_attribute(name, block)
    end
  end
end
```

Now calling `#first_name` and passing a block to an instance of `Factory` is equivalent to calling `#add_attribute` with `:first_name` and passing the block!

What is the `&` signed used for before the block? It is used to convert a block to a proc and vice versa. If you pass a `block` to a function, `&block` will convert it to a proc that you can store for later evaluation. Similarly, if you have a variable `my_proc` that stores a proc, `&my_proc` will convert the proc to a block. It's as simple as that!

This is what *syntactic sugar* is! You are only making the *API* simpler to remember and more elegant!

### Adding a factory registry

There is one last thing we need to do to complete our factory definition! We want to store all the defined factories in a registry to be able to retrieve them later based on their name. This will be useful when running one factory.

Let's add this feature very quickly when we define the Factory:

```rb
# lib/tiny_factory.rb

module TinyFactory
  @factories = []

  def self.factories
    @factories
  end

  def self.register_factory(factory)
    factories << factory
  end

  # Edit the .define method here
  def self.define(name, &block)
    factory = Factory.new(name)
    factory.instance_eval(&block)
    register_factory(factory)
    factory
  end

  # All the previous code
  # ...
end
```

Now all the factories that are defined are stored in the `TinyFactory.factories` array!

Wow! Nice work our Factory `:user` is now correctly defined!
Let's manually test it to see what happens by creating a `test.rb` file at the root of the project:

```rb
# test.rb

# This line enables you to require "tiny_factory" here
# You do not need to understand it
$LOAD_PATH.unshift File.expand_path("./lib", __dir__)
require "tiny_factory"

factory = TinyFactory.define :user do
  first_name { "Alexandre" }
  last_name { "Ruban" }
  email { "#{first_name}@hey.com".downcase }
end

p factory
```

If we then run this script in the console, here is what we get:

```bash
ruby test.rb

# =>
# #<TinyFactory::Factory:0x00007f93f305d000
#   @factory_name=:user,
#   @attributes=[
#     #<TinyFactory::Attribute:0x00007f93f305cd58
#       @name=:first_name,
#       @definition=#<Proc:0x00007f93f305ce20 test.rb:5>>,
#     #<TinyFactory::Attribute:0x00007f93f305cc90
#        @name=:last_name,
#        @definition=#<Proc:0x00007f93f305ccb8 test.rb:6>>,
#     #<TinyFactory::Attribute:0x00007f93f305cb28
#       @name=:email,
#       @definition=#<Proc:0x00007f93f305cb78 test.rb:7>>
#   ]>
```

As we can see, we get back a `Factory` object with a `factory_name` of `:user` and holding three attributes `:first_name`, `:last_name` and `:email`, each of them holding their `definition` proc!

That's a great achievement already! Now it's time for us to run the factory with one of the three methods `#attributes_for`, `#build` or `#create`!

## Running a Factory

Now that our `Factory` is defined, we need to run it. In our example it can mean three things:

- Return a hash of attributes (`#attributes_for`)
- Return a built instance (`#build`)
- Return a created instance persisted in the database (`#create`)

We will start by making `#attributes_for` work and then it will be very easy to add the `#build`and `#create`features.

### Making the attributes_for strategy work

For each of these three methods, we will define a strategy. Let's start with the `AttributesFor` strategy:

```rb
# lib/tiny_factory/strategy/attributes_for.rb

module TinyFactory
  class Strategy
    class AttributesFor
      def initialize
        @result = {}
      end

      def get(attribute)
        @result[attribute]
      end

      def set(attribute, value)
        @result[attribute] = value
      end

      def result
        @result
      end
    end
  end
end
```

The `AttributesFor` class is responsible for holding the result which is the hash of attributes that `attributes_for(:user)` returns in our integration test! What can we do with this class? We can get attributes from the result hash, set attributes to the result hash and, get the final result of our computation! Easy!

Now let's see how we can run this strategy! We will use `TinyFactory.attributes_for(:user)` for now instead of `attributes_for(:user)`. This *syntactic sugar* will be added at the end of the article!

Let's add the `TinyFactory.attributes_for` method:

```rb
# lib/tiny_factory.rb

# All the previous requires
#...
# /!\ Don't forget to add this new require
require "tiny_factory/strategy/attributes_for"

module TinyFactory
  # All the previous code
  # ...

  def self.attributes_for(factory_name)
    find_factory(factory_name).run(Strategy::AttributesFor)
  end

  def self.find_factory(factory_name)
    factories.find { |factory| factory.factory_name == factory_name }
  end
end
```

As you can see, we defined a `TinyFactory.attributes_for` method that looks for the factory by its name and the calls `#run` on it with the `AttributesFor` strategy class!

Let's add the `#run` method to the `Factory` class:

```rb
# lib/tiny_factory/factory.rb

module TinyFactory
  class Factory
    # All the previous code
    # ...

    def run(strategy_class)
      strategy = strategy_class.new
      @attributes.each do |attribute|
        attribute.add_to(strategy)
      end
      strategy.result
    end
  end
end
```

As you can see here, we create a new instance of the strategy class, then iterate through all the attributes of the factory and add each attribute to the strategy thanks to the `Attribute#add_to` method. Finally, we return the strategy result.

You guessed it, we are missing a `#add_to` method on the `Attribute` class. Let's add it:

```rb
# lib/tiny_factory/attribute.rb

module TinyFactory
  class Attribute
    def initialize(name, definition)
      @name = name
      @definition = definition
    end

    def add_to(strategy)
      # Not the final implementation
      strategy.set(@name, @definition.call)
    end
  end
end
```

Let's explain the `#add_to` method carrefully because there is a trick!

In our integration test example, our user factory has a `first_name` attribute and its definition is equivalent to `Proc.new { "Alexandre" }`. In this case `@definition.call` return `"Alexandre"`. That means that when we do `strategy.set(@name, @definition.call)` we are adding to the result hash, the key `:first_name` and the value `"Alexandre"`

Now let's take a look at our `email` attribute and its definition that is equivalent to `Proc.new { "#{first_name}@hey.com".downcase }`. This is more complicated because the `#first_name` method is not defined on the email attribute! How might we retrieve its value? We need to have a look at the strategy. But the strategy does not respond to the `first_name` method either. What are we going to use? You probably guessed it: `method_missing`!

```rb
# lib/tiny_factory/strategy/attributes_for.rb

module TinyFactory
  class Strategy
    class AttributesFor
    # All the previous code
    # ...

    def get(attribute)
      @result[attribute]
    end

    def method_missing(attribute)
      get(attribute)
    end
  end
end
```

Now our strategy responds to the `#first_name` method and returns the value that was set earlier on the result hash of the strategy instance.

The final implementation of the `#add_to` method is as follows:

```rb
# lib/tiny_factory/attribute.rb

module TinyFactory
  class Attribute
    def initialize(name, definition)
      @name = name
      @definition = definition
    end

    def add_to(strategy)
      strategy.set(@name, strategy.instance_eval(&@definition))
    end
  end
end
```

Notice the `&` before `@definition`? It converts the `@definition` proc to a bloc. Remember, the `&` sign before a block converts it to a `proc` and the `&` sign before a `proc` converts it back to a block, it's as simple as that!

We did it!

In the integration test, if you replace `#attributes_for(:user)` with `TinyFactory.attributes_for(:user)`, the test is green! We'll add the *syntactic sugar* to avoid having to write `TinyFactory` at the end of the article but that"s already a huge step forward!

### Making the build strategy work

To do this, we need to add a `TinyFactory::Strategy::Build`:

```rb
# lib/tiny_factory/strategy/build.rb

module TinyFactory
  class Strategy
    class Build
      # One small difference here, the strategy needs to be
      # initialized with a class. In our example, as we are building
      # a User instance, our build strategy will be initialized with the
      # User class.
      def initialize(klass)
        @instance = klass.new
      end

      def get(attribute)
        @instance.send(attribute)
      end

      def set(attribute, value)
        @instance.send("#{attribute}=", value)
      end

      def method_missing(attribute)
        get(attribute)
      end

      def result
        @instance
      end
    end
  end
end
```

Notice how it's almost the same as the `AttributesFor` strategy? The result of the strategy is not a `Hash` but an instance of the class passed to the `initialize` method. This means that to read an attribute, we need to send the attribute name to the instance, and to set an attribute with some value, we need to send the message `#{attribute}=` with the `value`.

As you noticed, we changed the number of arguments taken by the `initialize` method of a strategy. We want all our strategies to have the same *API* to make them interchangeable. This is known in Object-Oriented Programming as *polymorphism*. Let's change the `AttributesFor` strategy so that is also initialized with an argument:

```rb
module TinyFactory
  class Strategy
    class AttributesFor
      # We use an underscore here to indicate the argument passed is never used
      def initialize(_)
        @result = {}
      end

      # All the rest of the code
    end
  end
end
```

Even if we don't use the argument passed to initialize in the `AttributesFor` strategy, we still add it because we want to keep the same *API* between all of our strategies. This is called *polymorphism* and it's one of the most powerful features of OOP! If we didn't do that, we would need to check with `if` and `else` statements if the strategy we are using is `AttributesFor` or `Build`. This is called *type checking* and it's a code smell!

Let's go back to our `Build` strategy. We said that we are creating an instance of a class, but which class? By convention, when we create a factory with a name of `:user`, the class of the instance we build will be `User`! Let's add a tiny bit of code to create this convention:

```rb
# lib/tiny_factory/factory.rb

module TinyFactory
  class Factory
    # All the previous code

    def run(strategy_class)
      strategy = strategy_class.new(build_class)
      @attributes.each do |attribute|
        attribute.add_to(strategy)
      end
      strategy.result
    end

    private

    def build_class
      # This is why we need ActiveSupport as a dependency.
      # `classify` and `constantize` are Active Support methods.
      factory_name.to_s.classify.constantize
    end
  end
end
```

Thanks to `ActiveSupport#classify` and `ActiveSupport#constantize`, a factory with a name of `:user` will have a `build_class` of `User` and a factory with a name of `:billing_information` will have a `build_class` of `BillingInformation`. This is the convention we created! We guess the class to pass to the strategy from the name of the factory!

Wow! That's great progress!

One last thing to make it work, we need to require the file in our `tiny_factory.rb` and create the `.build` method.

```rb
# lib/tiny_factory.rb

# /!\ Add this require to the list
require "tiny_factory/strategy/build"

module TinyFactory
  # All the previous code

  def self.build(factory_name)
    find_factory(factory_name).run(Strategy::Build)
  end
end
```

You can now check that both the `#attributes_for` and the `#build` integration tests pass if you replace `attributes_for(:user)` with `TinyFactory.attributes_for(:user)` and `build(:user)` with `TinyFactory.build(:user)`!

### Making the create strategy work

This one is much much easier, we already did all the work! We simply need to add our `TinyFactroy::Strategy::Create`:

```rb
# lib/tiny_factory/strategy/create.rb

module TinyFactory
  class Strategy
    class Create < Build
      def result
        @instance.save!
        @instance
      end
    end
  end
end
```

The `Create` strategy is the same as the `Build` strategy except that we
need to `save!` the result instance in the database before we return it. To do this we make the `Create` strategy inherit from the `Build` strategy and simply override the `#result` method.

Once again, let's add the `TinyFactory.create` method and require the `create_strategy.rb` file:

```rb
# lib/tiny_factory.rb

# /!\ Add this require to the list
require "tiny_factory/strategy/create"

module TinyFactory
  # All the previous code
  # ...

  def self.create(factory_name)
    find_factory(factory_name).run(Strategy::Create)
  end
end
```

You can now make the create integration test pass by using `TinyFactory.create(:user)` instead of `create(:user)`.

You did it!

Let's now add one last piece of *syntactic sugar* and we will be done with this `FactoryBot` clone and you will know enough to dig through the real source code!

## The TinyFactory::Syntax::Default module

In our tests, we want to use `attributes_for(:user)` instead of `TinyFactory.attributes_for(:user)`, it is too verbose and we care about the syntax we will use every day as developers!

Let's add one last piece of *syntactic sugar* by creating a `TinyFactory::Syntax::Methods` module:

```rb
# lib/tiny_factory/syntax/methods.rb

module TinyFactory
  module Syntax
    module Methods
      def attributes_for(name)
        TinyFactory.attributes_for(name)
      end

      def build(name)
        TinyFactory.build(name)
      end

      def create(name)
        TinyFactory.create(name)
      end
    end
  end
end
```

The three methods created here only delegate to the `TinyFactory` module. This is very simple but will improve our experience as developers when we use the library!

To use this module we simply need to add this line in our test helper and to require the file in `tiny_factory.rb`:

```rb
# lib/tiny_factory.rb

# Add this require statement
require "tiny_factory/syntax/methods"
```

```rb
# test/test_helper.rb

# All the previous code
# ...
class Minitest::Test
  include TinyFactory::Syntax::Methods
end
```

How does this work? Our integration test inherits from `Minitest::Test`. By including our *syntactic sugar* module in `Minitest::Test` we automatically gain access to the `#attributes_for`, `#build` and `#create` methods that delegate to the `TinyFactory` module!

Let's run the integration test from the beginning of the article now. Everything should be green, you are now able to understand how `FactoryBot` works! Congratulations!

## Takeaways

In this articles, we learned:

- To create a gem with the `bundle gem` command
- To convert a block to a proc and vice versa with the `&` character
- To intercept class to undefined methods on an object thanks to `#method_missing`
- That *polymorphism* is an important concept in Object-Oriented Programming. If you want to learn more about the topic I highly recommend the [99 bottles of OOP](https://sandimetz.com/99bottles) book by Sandi Metz
- That it is possible to add *syntactic sugar* to your code to make it more elegant and easy to remember
- How the real FactoryBot works!

## Did you like this article?

If you liked this article and want to be notified when I publish more, **follow me** on [Twitter](https://twitter.com/alexandre_ruban)! Feel free to share this article with your developer friends and coworkers!
