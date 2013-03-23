SafeYAML
========

[![Build Status](https://travis-ci.org/dtao/safe_yaml.png)](http://travis-ci.org/dtao/safe_yaml)

The **SafeYAML** gem provides an alternative implementation of `YAML.load` suitable for accepting user input in Ruby applications. Unlike Ruby's built-in implementation of `YAML.load`, SafeYAML's version will not expose apps to arbitrary code execution exploits (such as [the ones discovered](http://www.reddit.com/r/netsec/comments/167c11/serious_vulnerability_in_ruby_on_rails_allowing/) [in Rails in early 2013](http://www.h-online.com/open/news/item/Rails-developers-close-another-extremely-critical-flaw-1793511.html)).

**If you encounter any issues with SafeYAML, check out the 'Commin Issues' section below.** If you don't see anything that addresses the problem you're experiencing, by all means, [create an issue](https://github.com/dtao/safe_yaml/issues/new)!

Installation
------------

Add this line to your application's Gemfile:

    gem "safe_yaml"

And then execute:

    $ bundle

Or install it yourself as:

    $ gem install safe_yaml

Configuration
-------------

Configuring SafeYAML should be quick. In most cases, you probably will only have to think about two things:

1. Do you want the `YAML` module's *default* behavior to allow or disallow arbitrary object deserialization? Set the `SafeYAML::OPTIONS[:default_mode]` option to either `:safe` or `:unsafe` to control this. If you do neither, SafeYAML will default to `:safe` mode but will issue a warning the first time you call `YAML.load`.
2. Do you want to allow symbol deserialization by default or not? Set the `SafeYAML::OPTIONS[:deserialize_symbols]` option to `true` or `false` to control this. The default is `false`, which means that SafeYAML will deserialize symbols in YAML documents as strings.

For more information on these and other options, see the "Usage" section below.

Explanation
-----------

Suppose your application were to use a popular open source library which contained code like this:

```ruby
class ClassBuilder
  def []=(key, value)
    @class ||= Class.new

    @class.class_eval <<-EOS
      def #{key}
        #{value}
      end
    EOS
  end

  def create
    @class.new
  end
end
```

Now, if you were to use `YAML.load` on user input anywhere in your application without the SafeYAML gem installed, an attacker who suspected you were using this library could send a request with a carefully-crafted YAML string to execute arbitrary code (yes, including `system("unix command")`) on your servers.

This simple example demonstrates the vulnerability:

```ruby
yaml = <<-EOYAML
--- !ruby/hash:ExploitableClassBuilder
"foo; end; puts %(I'm in yr system!); def bar": "baz"
EOYAML
```

    > YAML.load(yaml)
    I'm in yr system!
    => #<ClassBuilder:0x007fdbbe2e25d8 @class=#<Class:0x007fdbbe2e2510>>

With SafeYAML, the same attacker would be thwarted:

    > require "safe_yaml"
    => true
    > YAML.load(yaml, :safe => true)
    => {"foo; end; puts %(I'm in yr system!); def bar"=>"baz"}

Usage
-----

When you require the safe_yaml gem in your project, `YAML.load` is patched to accept one additional `options` parameter (for Syck and Psych prior to Ruby 1.9.3, this adds a second parameter; for Psych in 1.9.3 and later, it adds a third). The most important of these is the `:safe` option (default: `true`), which controls whether or not to deserialize arbitrary objects when parsing a YAML document. The other options, along with explanations, are as follows.

- `:deserialize_symbols` (default: `false`): Controls whether or not YAML will deserialize symbols. It is probably best to only enable this option where necessary, e.g. to make trusted libraries work. Symbols receive special treatment in Ruby and are not garbage collected, which means deserializing them indiscriminately may render your site vulnerable to a DOS attack (hence the default of `false`).

- `:whitelisted_tags`: Provides a list of YAML tags that designate trusted types, e.g., ones that can be deserialized without worrying about any resulting security vulnerabilities. When any of the given tags are encountered in a YAML document, the associated data will be parsed by the underlying YAML engine (Syck or Psych) for the version of Ruby you are using. See "Whitelisting Trusted Types" below for more information.

- `:custom_initializers`: Similar to `whitelisted_tags`, but allows you to provide your own initializers for specified tags rather than using Syck or Psyck.

- `:raise_on_unknown_tag` (default: `false`): This option represents the highest possible level of paranoia (not necessarily a bad thing); if the YAML engine encounters any tag other than ones that are automatically trusted by SafeYAML or that you've explicitly whitelisted, it will raise an exception. This may be a good choice if you expect to always be dealing with perfectly safe YAML and want your application to fail loudly upon encountering questionable data.

All of the above options can be set at the global level via `SafeYAML::OPTIONS`. You can also set each one individually per call to `YAML.load`; an option explicitly passed to `load` will take precedence over an option specified globally.

Supported Types
---------------

The way that SafeYAML works is by restricting the kinds of objects that can be deserialized via `YAML.load`. More specifically, only the following types of objects can be deserialized by default:

- Hashes
- Arrays
- Strings
- Numbers
- Dates
- Times
- Booleans
- Nils

Again, deserialization of symbols can be enabled globally by setting `SafeYAML::OPTIONS[:deserialize_symbols] = true`, or in a specific call to `YAML.load([some yaml], :deserialize_symbols => true)`.

Whitelisting Trusted Types
--------------------------

SafeYAML supports whitelisting certain YAML tags for trusted types. This is handy when your application uses YAML to serialize and deserialize certain types not listed above, which you know to be free of any deserialization-related vulnerabilities.

The easiest way to whitelist types is by calling `SafeYAML.whitelist!`, which can accept a variable number of safe types, e.g.:

```ruby
SafeYAML.whitelist!(FrobDispenser, GobbleFactory)
```

You can also whitelist YAML *tags* via the `:whitelisted_tags` option:

```ruby
# Using Syck
SafeYAML::OPTIONS[:whitelisted_tags] = ["tag:ruby.yaml.org,2002:object:OpenStruct"]

# Using Psych
SafeYAML::OPTIONS[:whitelisted_tags] = ["!ruby/object:OpenStruct"]
```

And in case you were wondering: no, this feature will *not* allow would-be attackers to embed untrusted types within trusted types:

```ruby
yaml = <<-EOYAML
--- !ruby/object:OpenStruct 
table: 
  :backdoor: !ruby/hash:ExploitableClassBuilder 
    "foo; end; puts %(I'm in yr system!); def bar": "baz"
EOYAML
```

    > YAML.safe_load(yaml)
    => #<OpenStruct :backdoor={"foo; end; puts %(I'm in yr system!); def bar"=>"baz"}>

Known Issues
------------

If you add SafeYAML to your project and start seeing any errors about missing keys, or you notice mysterious strings that look like `":foo"` (i.e., start with a colon), it's likely you're seeing errors from symbols being saved in YAML format. If you are able to modify the offending code, you might want to consider changing your YAML content to use basic strings instead of symbols. If not, you may need to set the `:deserialize_symbols` option to `true`, either in calls to `YAML.load` or--as a last resort--globally, with `SafeYAML::OPTIONS[:deserialize_symbols]`.

Also be aware that some Ruby libraries, particularly those requiring inter-process communication, leverage YAML's object deserialization functionality and therefore may break or otherwise be impacted by SafeYAML. The following list includes known instances of SafeYAML's interaction with other Ruby gems:

- [**ActiveRecord**](https://github.com/rails/rails/tree/master/activerecord): uses YAML to control serialization of model objects using the `serialize` keyword. If you find that accessing serialized properties on your ActiveRecord models is causing errors, chances are you may need to (a) set the `:deserialize_symbols` option to `true`, (b) whitelist some of the types in your serialized data via `SafeYAML.whitelist!` or the `:whitelisted_tags` option, or (c) both.
- [**Guard**](https://github.com/guard/guard): Uses YAML as a serialization format for notifications. The data serialized uses symbolic keys, so setting `SafeYAML::OPTIONS[:deserialize_symbols] = true` is necessary to allow Guard to work.
- [**sidekiq**](https://github.com/mperham/sidekiq): Uses a YAML configiuration file with symbolic keys, so setting `SafeYAML::OPTIONS[:deserialize_symbols] = true` should allow it to work.

The above list will grow over time, as more issues are discovered.

Caveat
------

This gem is quite young, and so the API may (read: *will*) change in future versions. The goal of the gem is to make it as easy as possible to protect existing applications from object deserialization exploits. Any and all feedback is more than welcome.

Requirements
------------

SafeYAML requires Ruby 1.8.7 or newer and works with both [Syck](http://www.ruby-doc.org/stdlib-1.8.7/libdoc/yaml/rdoc/YAML.html) and [Psych](http://github.com/tenderlove/psych).

If you are using a version of Ruby where Psych is the default YAML engine (e.g., 1.9.3) but you want to use Syck, be sure to set `YAML::ENGINE.yamler = "syck"` **before** requiring the safe_yaml gem.
