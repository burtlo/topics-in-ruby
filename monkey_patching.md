# Monkey Patching

## An Angel

A gem you found solves 90% of what you want it to do. It falls short in one
particular, important case. You track down the deficiency and write a quick fix.
You make a mental note to extract your change from the project and submit it
back to the gem author when you eventually come up for air.

## A Devil

While running your application you start to notice a peculiar behavior: all the
text in the output is appearing capitalized. An initial look through your code
fails to show anything wrong. It is only after you start debugging you notice
that the `String#capitalize` method is uppercasing the first letter of every
word in a string. Searching through Ruby documentation you realize that
`String#capitalize` should only be uppercasing the first letter of the first
word. What happened?

Scouring through the code again you find the previous maintainer had decided to re-implement `String#capitalize`; giving the method it's new meaning.

## Changing Expectations

As developers we have expectations of how the language, frameworks and libraries
we use works. We depend on it. So when a system starts to challenge those
expectations we often become very confused.

A ruby developer is allowed to add or redefine methods on an existing object.
This is called monkey patching. Monkey patching as a practice carries the
*possibility* of causing all kinds of untold damage and confusion.

Within this section I outline how to monkey patch while providing tactics on
doing it responsibly.

## What is Monkey Patching?

First and foremost it is important to know how to monkey patch. This is the
example code snippet that caused the problem in the introduction:

```ruby
class String
  def capitalize
    self.gsub(/^[a-z]|\s+[a-z]/) { |a| a.upcase }
  end
end
```

Ruby's open classes make it incredibly easy to monkey patch. This small snippet
of could live within your own project or in one of project's dependent gems.
Once this code is executed, it will change the behavior of `String#capitalize`
for the entire execution of the application (or until it is changed again).

IRB allows us to quickly verify this behavior:

```
:001 > "a good working relationship".capitalize
=> "A good working relationship"
:002 > class String
:003?>   def capitalize
:004?>     self.gsub(/^[a-z]|\s+[a-z]/) { |a| a.upcase }
:005?>   end
:006?> end
 => nil
:007 > "a good working relationship".capitalize
 => "A Good Working Relationship"
```

Opening IRB in another terminal you will find `#capitalize` behaves as it did
originally.

Take a moment to admire the power has given you. This is something you would
never be able to accomplish in many other programming language.

## Know Before You Patch

The developer monkey patching `String` may have not intended to redefine
`#capitalize`. They may not have realized that capitalize already existed. How
could they have known they were going to get in trouble?

> You as a reader may be wondering how this could happen. If they had tried to execute the method before they went to straight to implementing it, they would have been seen that it already existed. Perhaps they had made an attempt to use the method but with a spelling error.

### Using Documentation

[Rubydoc.info](http://www.rubydoc.info/stdlib) is an invaluable resource when it
comes to providing a browsable and searchable resource for classes and methods.
Browsing through [String](http://www.rubydoc.info/stdlib/core/String) you will find an entry for [capitalize](http://www.rubydoc.info/stdlib/core/String#capitalize-instance_method) as well as
[capitalize!](http://www.rubydoc.info/stdlib/core/String#capitalize%21-instance_method).

### #instance_methods and #methods

Documentation does not represent the current state of the code within the
context of your application. Fortunately Ruby allows you to query the methods of
an object and the instance methods of a class or module. This practice is
commonly referred to as introspection.

Every class and module will respond to
[#instance_methods](http://www.rubydoc.info/stdlib/core/Module#instance_methods-instance_method).
`#instance_methods` will return all the methods that an instance of `String`
will have when it is created. This is the exact same list of methods that you
would find as a result of the methods `methods` for a instance of a `String`.

```ruby
"a string object".methods # => [ :=>, :eql?, :hash, ... ]
"a string object".class   # => String
String.instance_methods   # => [ :=>, :eql?, :hash, ... ]
```

This knowledge could be used to your advantage to prevent an unintended monkey patch:

```ruby
unless String.instance_methods.include? :capitalize

  class String
    def capitalize
      self.gsub(/^[a-z]|\s+[a-z]/) { |a| a.upcase }
    end
  end

end
```

### Test Driven Development (TDD)

This is also a situation that TDD would have likely caught.

```ruby
describe String do
  let(:subject) { "a good working relationship" }
  its(:capitalize) { should == "A Good Working Relationship" }
end
```

This following test will fail with an expectation that the given string does not
match the expected string.

```
Failure/Error: its(:capitalize) { should == "A Good Working Relationship" }
       expected: "A Good Working Relationship"
            got: "A good working relationship" (using ==)
```

If the original developer did not know about the existing method they would have
seen the above failed expectation when they were likely expecting:

```
 Failure/Error: its(:capitalize) { should == "A Good Working Relationship" }
 NoMethodError:
   undefined method `capitalize' for "a good working relationship":String
```

This would raise the question. How is it that this method exists when I have not
written a single line of code to support it?

### defined?

We assumed the original developer intended only to define a new method. It is
possible they had intended to define a new class. Defining a new class and
appending additional methods to an existing class use the same notation.

While you are likely not going to make that mistake with `String` and other core
classes, there is always the possibility that might happen. Checking that a
class or module exists before you implement ensures clarity of intention and
provides insurance against unintended functionality.

Class names and module names are constants. When you create a class or a module
you defining a constant. You can determine if a constant is defined by using the
`defined?` method. `defined?` returns true if the constant is defined and false
if the constant is not defined.

If your intent is to monkey patch an existing class you would write:

```ruby
if defined? String

  # Defining or redefining methods on the existing String class

  class String
    def capitalize
      self.gsub(/^[a-z]|\s+[a-z]/) { |a| a.upcase }
    end
  end

end
```

If you intent is to define a new class you would write:

```ruby
unless defined? String

  # Defining the String class if one has not already been defined.

  class String
    def capitalize
      self.gsub(/^[a-z]|\s+[a-z]/) { |a| a.upcase }
    end
  end

end
```

## Safer Monkey Patching

### Unique Naming

First and likely the simplest solution is to solve the possible collision of
methods is to select a unique method name.

> If you want to be absolutely sure you are not re-implementing a method
> that already exists it is important that you employ `respond_to?` to
> ensure you have not done any wrong.

Defining the method with an explicit name brings clarity to the reader and
increases the likelihood that it will not collide with an existing method:

```ruby
class String
  def capitalize_all_first_letters
    self.gsub(/^[a-z]|\s+[a-z]/) { |a| a.upcase }
  end
end
```

> ActiveSupport’s inflector provides similar functionality called `#titleize`.
Developers often debate if it is worth adding ActiveSupport to a project if you
are using it for one such helper method. One benefit is that most Ruby
developers are familiar with Rails and the additional methods that ActiveSupport
provides. Making it a more standard choice.

You may also preface or suffix a method with a unique identifier related to the
project, your name or your organization.

```ruby
class String
  def organization_capitalize
    self.gsub(/^[a-z]|\s+[a-z]/) { |a| a.upcase }
  end
end
```

### Add an optional parameter in the re-implementation

Another possibility is to re-implement the method with the same name except add
an additional, optional parameter, that you can you provide in the instances
when you want to use the new behavior.

```ruby
class String
  def capitalize(all_words = false)
    regex_will_match = (all_words ? /^[a-z]|\s+[a-z]/ : /^[a-z]/)
    self.gsub(regex_will_match) { |a| a.upcase }
  end
end
```

This will preserve all existing uses of `String#capitialize` while providing the
ability to access the new functionality.

```ruby
"these words are important".capitalize # => "These words are important"
"these words are important".capitalize(true) # => "These Words Are Important"
```

Though this implementation gets the job done, the new parameter does not clearly
state it's purpose to those reading the code. This would be clearer:


```ruby
"these words are important".capitalize(:all_words => true) # => "These Words Are Important"
```
A named parameter would make the intent of this additional parameter more clear.

Here is an implementation that employs named parameters:

```ruby
class String
  def capitalize(options = {})
    options = { :all_words => false }.merge options
    regex_to_match = (options[:all_words] ? /^[a-z]|\s+[a-z]/ : /^[a-z]/)
    self.gsub(regex_to_match) { |a| a.upcase }
  end
end
```

Most developers would be satisfied with this implementation. However, it is
important to be aware that we re-implemented the original `#capitalize` method
within our new `capitalize`. While the implementation appears sound it is not
the original implementation.

> Also consider the situation where the next version of Ruby changes the functionality of the original method. This means our overridden method would default would be incorrect.

Ideally our new method would call the original method when used the default way.
Saving us from re-implementing the original method and preventing issues if the
original should purposively change.

### Aliasing a Method

An incredibly awesome utility at your disposal is
[Module#alias_method](http://rubydoc.info/stdlib/core/1.9.3/Module:alias_method)
. `#alias_method` creates a copy of an existing method and assigns it a new
name.

```ruby
alias_method new_name_for_method, original_method_name
```

Within our new `#capitalize` method we can call the code of the original
capitalize that we have preserved with the new method name
`#original_capitalize`.

This is an implementation using `#alias_method`:

```ruby
class String
  alias_method :original_capitalize, :capitalize

  def capitalize(options = {})
    options = { :all_words => false }.merge options

    if options[:all_words]
      self.gsub(/^[a-z]|\s+[a-z]/) { |a| a.upcase }
    else
      original_capitalize
    end
  end

end
```

> Ruby has a keyword `alias`, but it is not as flexible as  `Module#alias_method`.

* http://stackoverflow.com/questions/4763121/ruby-should-i-use-alias-or-alias-method
* http://blog.bigbinary.com/2012/01/08/alias-vs-alias-method.html

There are two caveats to this solution:

First, when selecting the name for the new alias method, we could have
inadvertently overwritten an existing method. Though, we could employ the
previously outlined tactics to prevent unintended consequences.

Second, `String` now responds to `#capitalize` and `#original_capitalize`. Any
code that relies on the stability of a class's instance methods may be affected
by this change.

Despite these two wrinkles I would consider aliasing an acceptable solution.

### An alternative to Aliasing

If those costs of aliasing leaves a bad taste in your mouth there is
alternative.

With aliasing we renamed the original method and then called it. Instead of
copying this method to a new name what we really want is to do the following:

* Retrieve the original method by name
* Keep a copy of the original method
* Define the replacement method
* For the default case within the new method execute the original method

This can be all be done in Ruby with introspection and meta-programming.

> Thanks to [Jay
Fields](http://blog.jayfields.com/2006/12/ruby-alias-method-alternative.html)
for reposting [Martin Traverson](http://split-s.blogspot.com/2006/01/replacing-methods.html)'s solution.

#### Retrieving the original method

First we need to retrieve the method. Module provides the method
[instance_method](http://www.rubydoc.info/stdlib/core/Module#instance_method-instance_method)
which accepts the name of the method as a parameter and returns an instance of
[UnboundMethod](http://rubydoc.info/stdlib/core/1.9.3/UnboundMethod).

```ruby
capitalize = String.instance_method(:capitalize)

puts capitalize # => <UnboundMethod: String#capitalize>
```

We can also retrieve it from within the class itself:

```
class String
  capitalize_method = self.instance_method(:capitalize)
  puts capitalize_method # => <UnboundMethod: String#capitalize>
end
```

Why does it return an
[UnboundMethod](http://rubydoc.info/stdlib/core/1.9.3/UnboundMethod) and not a
[Method](http://rubydoc.info/stdlib/core/1.9.3/Method)?

The list of instance methods maintained by a module are bound to an *instance
of the module* and not the module. A quick demonstration of this can be seen
by running the following:

```ruby
puts String.instance_method(:capitalize) # => <UnboundMethod: String#capitalize>
puts "Try me!".method(:capitalize) # => #<Method: String#capitalize>
```

#### Defining the replacement method

`Module#define_method` allows you to define a method with a block:

```ruby
class String
  capitalize_method = self.instance_method(:capitalize)

  define_method(:capitalize) do |options = {}|
    # ... new capitalize implementation
  end

end
```

Here we are defining a new `capitalize` method which accepts our hash of named
parameters; defaulting when no parameters are specified. We want to use
`#define_method` here because the variable `capitalize_method` is still in scope
allowing us to reference it. Producing the following:

```ruby
class String
  capitalize_method = self.instance_method(:capitalize)

  define_method(:capitalize) do |options = {}|
    options = { :all_words => false }.merge options

    if options[:all_words]
      self.gsub(/^[a-z]|\s+[a-z]/) { |a| a.upcase }
    else
      capitalize_method.bind(self).call
    end
  end

end
```

`UnboundMethod` instances can be bound to any object. In this case we are
binding it back to the String instance. If it is not bound to an object it
simply cannot be called.

## Where to keep your Monkey Patches

So far we have talked about various implementation details about monkey
patching. What remains is where within your application is the best place to define your monkey patches.

### Including a Module

Placing your implementation of `#capitalize` within a custom module would allow
you to apply that functionality to more than one class while also making your
intentions more clear to other readers.

```ruby
module CustomCapitalize
  def capitalize
    self.gsub(/^[a-z]|\s+[a-z]/) { |a| a.upcase }
  end
end

class String
  include CustomCapitalize
end

"these words are important".capitalize # => "These words are important"
```

Unfortunately, this does not work!

When the `CustomCapitalize` module is included in the `String` class it **will
not** override the existing methods with the same name. That is because the
newly included module is not placed immediately within the String's object
hierarchy but one level above.

You can see what is happening by looking at String's ancestors:

```ruby
module CustomCapitalize
  def capitalize
    self.gsub(/^[a-z]|\s+[a-z]/) { |a| a.upcase }
  end
end

class String
  include CustomCapitalize
end

String.ancestors # => [String, CustomCapitalize, Comparable, Object, Kernel, BasicObject]
```

During execution Ruby will look first for `String#capitalize` and only if it is
not present there will it proceed to look further up the chain at
`CustomCapitalize#capitalize`.

You could use this to your advantage if you want to ensure that you do not
accidentally monkey patch a class. However, I would suggest against this
implementation because it would likely confuse another reader.

### The file for your monkey patch

It is important to keep your monkey patches in a location that allows a reader
to quickly surmise what has been monkey patched.

A monkey patch should be maintained in a file that matches the name of the class
you are changing. And that file should be stored in the directory, with an
*_ext* suffix, named after the gem or the location with in the Ruby library.

As `String` resides within Ruby's core library I would store `String#capitalize`
at `lib/core_ext/string.rb`.

## Summary

Monkey patching is a powerful tool at your disposal. The ability to quickly add
functionality or quickly replace existing functionality that may be broken or
not appropriate for the environment you wish to execute within is undoubtably a
great boon.

But also remember, monkey patching can also be a devil. Changing the fundamental
expectations of the language is like changing the rules of the game after it has
started.

Use it wisely.