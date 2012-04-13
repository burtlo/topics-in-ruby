# Monkey Patching

## An Angel

A gem you found solves 90% of what you want it to do. It falls short in one
particular, important case. You quickly track down the deficiency and write a
quick patch that you leave within the project. With a note to yourself to open
up a pull request.

## A Devil

While running your Rails application you start to notice some peculiar behavior
where all the words in a particular user entered text are appearing in your
database with the first letter of each word capitalized. An initial look from
your client-side code all the way through to your database models all look
correct. Your tests all pass. It is only when you start debugging that you see
that all of sudden the capitalize method is no longer simply uppercasing the
first letter of the first word but uppercasing the first letter of every word.
What happened?

Scouring through your code you find the previous maintainer had decided at some
point found it useful to give capitalize its new meaning.

## Changing Expectations

As developers we develop expectations of how the language and particularly the
frameworks and libraries work. We depend on it. So when a system starts to
challenge those expectations we often become very confused.

That is why I think most developers live in fear of Ruby’s monkey patching. At
least, that is the expectations painted by the few outspoken developers blogging
about all the dangers of Ruby’s open classes and monkey patching.

Monkey patching gets a bad name because as a practice it carries the possibility
of causing all kinds of untold damage and confusion. However, I feel as though a
lot of actions within the software development carry that same possibility.

This section tries to show you how to monkey patch and some tactics for
attempting to do it safely.

## Understand the Problem

First and foremost it is important to know how to monkey patch. It is only then
can you learn to how to find when it has been done and choose safer or
alternative solutions.

```ruby
class String
  def capitalize
    self.gsub(/^[a-z]|\s+[a-z]/) { |a| a.upcase }
  end
end
```

In Ruby it is incredibly easy to monkey patch, hence the constant worry, as
Ruby’s classes are regarded as being Open Classes. This small snippet of code
that defines a new capitalize can exist within any source file within your
project or one of the gems that you use within your project. Simply by running
this small snippet of code, you change the behavior of String for the entire
execution of the application (or until it is changed again).

You can quickly verify this by launching IRB, using the original capitalize,
copy & pasting this small sample of code, and then using capitalize again.

```
:001 > "a good working relationship".capitalize
=> "A good working relationship" 
:002 > class String
:003?>     def capitalize
:004?>         self.gsub(/^[a-z]|\s+[a-z]/) { |a| a.upcase }
:005?>       end
:006?>   end
 => nil 
:007 > "a good working relationship".capitalize
 => "A Good Working Relationship" 
```

Leaving IRB and returning you will find that all has returned to normal and
capitalize is once again returned to its former self.

If you have every tried to do something like this in another programming
language, you were likely shutdown. Ruby provides you with the awesome power to
redefine how its core classes work. This is awesome.

## Know Before You Patch

The previous capitalize example could have been a mistake by the original
developer. Perhaps they never intended to overwrite the capitalize method. How
could they have known they were going to get in trouble?

### Using Documentation

Rubydoc.info is an invaluable resource when it comes to providing a browsable
and searchable resource for classes and methods. Browsing through `String` you
will find an entry for `capitalize` as well as `capitalize!`.

### instance_methods && methods

Documentation does not represent the current state of the code within the
context of your application. Fortunately Ruby allows you to query the methods of
an object and the instance methods of a class or module. This practice is
commonly referred to as introspection.

```ruby
"a string object".methods # => [ :=>, :eql?, :hash, ... ]
"a string object".class   # => String
String.instance_methods   # => [ :=>, :eql?, :hash, ... ]
```

Every class and module will respond to `instance_methods`. `instance_methods`
will return all the methods that an instance of `String` will have when it is
created.

```ruby
unless String.instance_methods.include? :capitalize

  class String
    def capitalize
      self.gsub(/^[a-z]|\s+[a-z]/) { |a| a.upcase }
    end
  end

end
```

This is the exact same list of methods that you would find as a result of the
methods `method` for a instance of a `String`.

### Test Driven Development (TDD)

This is also a situation that TDD would have likely caught.  

```ruby
describe String do
  let(:subject) { "a good working relationship" }
  its(:capitalize) { should == "A Good Working Relationship" }
end
```

This following test will fail but with an error that shows `String` already has
a `capitalize` method defined. Albeit a different one from the expectations. But
an unexpected error none the less for having not written a single line of
`capitalize` code.

```
Failure/Error: its(:capitalize) { should == "A Good Working Relationship" }
       expected: "A Good Working Relationship"
            got: "A good working relationship" (using ==)
```

### defined?

We have been making an assumption that the original developer intended only to
define an additional method. Perhaps they did not know that String existed and
intended to define a new class called `String`. Granted this is likely not the
case based on the implementation chosen by the developer.

    > Defining a new class or appending additional methods to an existing class
    > uses the same notation so there is no way to understand intention from
    > reading the code.

While you are likely not going to make that mistake with `String` and other core
classes, there is always the possibility that might happen. Checking that a
class or module exists before you implement ensures clarity of intention and
provides insurance against unintended functionality.

The only way to ensure you are either defining a new class or appending methods
can be guaranteed through the defined? operator.

`defined?` returns true if the constant is defined and false if the constant is
not defined. When you create a class or a module you defining a constant. Prior
to the initial declaration the constant has not been defined.

In this example we are ensuring not to define `String` if the class has already
been defined somewhere else.

```ruby
unless defined? String

  # Preventing a monkey-patching

  class String
    def capitalize
      self.gsub(/^[a-z]|\s+[a-z]/) { |a| a.upcase }
    end
  end

end
```

However, if you wanted to ensure that you were monkey patching you could have
used the inverse.

```ruby
if defined? String

  # Knowingly monkey-patching

  class String
    def capitalize
      self.gsub(/^[a-z]|\s+[a-z]/) { |a| a.upcase }
    end
  end

end
```

You likely do not write code this way unless you are extremely paranoid. It
would become tedious and ruinous to the flexibility of the Ruby code that you
write, so you often rely on your understanding of Ruby to ensure you are not
stepping on its toes.

## Addressing the Problem

### Unique Naming

First and likely the simplest solution to solve the possible collision of 
methods is to select a unique method name.

    > If you want to be absolutely sure you are not re-implementing a method
    > that already exists it is important that you employ `respond_to?` to 
    > ensure you have not done any wrong.

Defining the method with a very explicit name has the benefit of giving more 
clarity to those reading the code. It will likely clue them in that this 
method is monkey patched.

```ruby
class String
  def capitalize_all_first_letters
    self.gsub(/^[a-z]|\s+[a-z]/) { |a| a.upcase }
  end
end
```

    > ActiveSupport supplies a similar method to String named `titleize`.

You may also preface or suffix a method with a unique identifier related to the
project, your name or your organization.

```ruby
class String
  def organization_capitalize
    self.gsub(/^[a-z]|\s+[a-z]/) { |a| a.upcase }
  end
end
```

### Add a parameter to the method in the re-implementation

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

"these words are important".capitalize # => "These words are important"
"these words are important".capitalize(true) # => "These Words Are Important"
```

This will preserve all existing uses of `capitialize` while providing the 
ability to access the new functionality.

Though this implementation gets the job an implementation that leads towards 
named parameters would likely be more clear.

```ruby
class String
  def capitalize(options = {})
    options = { :all_words => false }.merge options
    regex_to_match = (options[:all_words] ? /^[a-z]|\s+[a-z]/ : /^[a-z]/)
    self.gsub(regex_to_match) { |a| a.upcase }
  end
end

"these words are important".capitalize(:all_words => true) # => "These Words Are Important"

```

There are some concerns with this course of action though and that is I have 
re-implemented the default capitalize method. And while the output of this 
method is well known and the implementation is fairly sound it is not the 
original implementation.

    > Consider the situation where the next version of Ruby changes the >
    functionailty of `capitalize`. This overriden method defaults now to an >
    incorrect implementation.
    
So, while a desirable monkey-patching option it is not truly sound unless we
were able to create this new method while still having a reference to the
original method.

### Alias Method

Preserving the original goal should be paramount if you want to be clever with
your monkey-patching and that is solely because you will likely introduce harder
to detect errors when you are executing this code or code that depends on this
code.

An incredibly awesome utility at your disposal is `Modules` 
[alias_method](http://rubydoc.info/stdlib/core/1.9.3/Module:alias_method) which 
allows you to create a copy of an existing method with a new name. Essentially
freeing you up to define a new method with the original name and being able to
call the original method

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

We create a copy, through alias_method, of the `capitalize` method. We call the
copy of the original `original_capitalize`. 

    > alias_method NEW_NAME_FOR_METHOD, ORIGINAL_METHOD

The reason for that is because we immediately define a new `capitalize` method
which is able to call the copied method, `original_capitalize` when the user 
has not provided any parameter.

Ruby has a keyword `alias`, but it is far less flexible as `alias_method`.

* http://stackoverflow.com/questions/4763121/ruby-should-i-use-alias-or-alias-method
* http://blog.bigbinary.com/2012/01/08/alias-vs-alias-method.html

When selecting a name for the aliased method, we could have inadvertently
overwritten an existing method with that name. Again, you could rely on any
of the previously outlined tactics for ensuring you have not mistakenly
overridden a method.

Also, an unintended effect, that may cause problems, is that the class `String` 
now has a new instance method `original_capitalize` that is an available to be
called. So if any code relies on the stability of the available instance methods
consider this a major problem.
  
### An alternative to Aliasing

If the unintended effect of the additionally generated method leaves a bad
taste in your mouth there is another alternative that 
[Jay Feilds](http://blog.jayfields.com/2006/12/ruby-alias-method-alternative.html) 
outlines which employs a number of meta-programming techniques to solve the 
problem.

```ruby
class String
  capitalize = self.instance_method(:capitalize)

  define_method(:capitalize) do |options = {}|
    options = { :all_words => false }.merge options
    
    if options[:all_words]
      self.gsub(/^[a-z]|\s+[a-z]/) { |a| a.upcase }
    else
      capitalize.bind(self).call
    end
    
  end

end
```

### include Module (will not override)

### Override give warnings, output error

## Layout of the Code

### Place within core_ext
