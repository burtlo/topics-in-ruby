# Monkey Patching

## An Angel

A gem you found solves 90% of what you want it to do. It falls short in one particular, important case. You quickly track down the deficiency and write a quick patch that you leave within the project. With a note to yourself to open up a pull request.

## A Devil

While running your Rails application you start to notice some peculiar behavior where all the words in a particular user entered text are appearing in your database with the first letter of each word capitalized. An initial look from your client-side code all the way through to your database models all look correct. Your tests all pass. It is only when you start debugging that you see that all of sudden the capitalize method is no longer simply uppercasing the first letter of the first word but uppercasing the first letter of every word. What happened?

Scouring through your code you find the previous maintainer had decided at some point found it useful to give capitalize its new meaning.

## Changing Expectations

As developers we develop expectations of how the language and particularly the frameworks and libraries work. We depend on it. So when a system starts to challenge those expectations we often become very confused. 

That is why I think most developers live in fear of Ruby’s monkey patching. At least, that is the expectations painted by the few outspoken developers blogging about all the dangers of Ruby’s open classes and monkey patching.

Monkey patching gets a bad name because as a practice it carries the possibility of causing all kinds of untold damage and confusion. However, I feel as though a lot of actions within the software development carry that same possibility.

This section tries to show you how to monkey patch and some tactics for attempting to do it safely.

## Understand the Problem

First and foremost it is important to know how to monkey patch. It is only then can you learn to how to find when it has been done and choose safer or alternative solutions.

```ruby
class String
  def capitalize
    self.gsub(/^[a-z]|\s+[a-z]/) { |a| a.upcase }
  end
end
```

In Ruby it is incredibly easy to monkey patch, hence the constant worry, as Ruby’s classes are regarded as being Open Classes. This small snippet of code that defines a new capitalize can exist within any source file within your project or one of the gems that you use within your project. Simply by running this small snippet of code, you change the behavior of String for the entire execution of the application (or until it is changed again).

You can quickly verify this by launching IRB, using the original capitalize, copy & pasting this small sample of code, and then using capitalize again.

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

Leaving IRB and returning you will find that all has returned to normal and capitalize is once again returned to its former self.

If you have every tried to do something like this in another programming language, you were likely shutdown. Ruby provides you with the awesome power to redefine how its core classes work. This is awesome.

## Know Before You Patch

The previous capitalize example could have been a mistake by the original developer. Perhaps they never intended to overwrite the capitalize method. How could they have known they were going to get in trouble?

### Using Documentation

Rubydoc.info is an invaluable resource when it comes to providing a browsable and searchable resource for classes and methods. Browsing through String you will find an entry for capitalize as well as capitalize!.

### instance_methods && methods

Documentation does not represent the current state of the code within the context of your application. Fortunately Ruby allows you to query the methods of an object and the instance methods of a class or module. This practice is commonly referred to as introspection.

```ruby
"a string object".methods # => [ :=>, :eql?, :hash, ... ]
"a string object".class   # => String
String.instance_methods   # => [ :=>, :eql?, :hash, ... ]
```

Every class and module will respond to instance_methods. instance_methods will return all the methods that an instance of String will have when it is created.

```ruby
unless String.instance_methods.include? :capitalize

  class String
    def capitalize
      self.gsub(/^[a-z]|\s+[a-z]/) { |a| a.upcase }
    end
  end

end
```

This is the exact same list of methods that you would find as a result of the methods method for a instance of a String.

### Test Driven Development (TDD)

This is also a situation that TDD would have likely caught.  

```ruby
describe String do
  let(:subject) { "a good working relationship" }
  its(:capitalize) { should == "A Good Working Relationship" }
end
```

This following test will fail but with an error that shows String already has a capitalize method defined. Albeit a different one from the expectations. But an unexpected error none the less for having not written a single line of capitalize code.

```
Failure/Error: its(:capitalize) { should == "A Good Working Relationship" }
       expected: "A Good Working Relationship"
            got: "A good working relationship" (using ==)
```

### defined?

We have been making an assumption that the original developer intended only to define an additional method. Perhaps they did not know that String existed and intended to define a new class called String. Granted this is likely not the case based on the implementation chosen by the developer.

    > Defining a new class or appending additional methods to an existing class uses the same notation so there is no way to understand intention from reading the code. 

While you are likely not going to make that mistake with String and other core classes, there is always the possibility that might happen. Checking that a class or module exists before you implement ensures clarity of intention and provides insurance against unintended functionality.
    
The only way to ensure you are either defining a new class or appending methods can be guaranteed through the defined? operator.

defined? returns true if the constant is defined and false if the constant is not defined. When you create a class or a module you defining a constant. Prior to the initial declaration the constant has not been defined.

In this example we are ensuring not to define String if the class has already been defined somewhere else.

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

However, if you wanted to ensure that you were monkey patching you could have used the inverse.

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

You likely do not write code this way unless you are extremely paranoid. It would become tedious and ruinous to the flexibility of the Ruby code that you write, so you often rely on your understanding of Ruby to ensure you are not stepping on its toes.
