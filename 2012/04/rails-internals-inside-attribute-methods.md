# Rails Internals: Inside Active Model Attribute Methods

Attribute methods is a useful class that alllows you to define dynamic
prefixes/suffixes of your attributes:

``` ruby
require 'rubygems'
require 'active_model'

class Person
  include ActiveModel::AttributeMethods

  attr_accessor :name

  attribute_method_affix :prefix => 'reset_', :suffix => '_to_default'
  attribute_method_suffix '_present?'
  attribute_method_suffix '_contains?'
  define_attribute_methods %w{name}

  private

  def reset_attribute_to_default(attr)
    send("#{attr}=", nil)
  end

  def attribute_present?(attr)
    !send(attr).nil?
  end

  def attribute_contains?(attr, substring)
    send(attr).include?(substring)
  end

end

p = Person.new
p.name = "Oscar"
p p.name_present? # true
p p.name_contains?("Os") # true
p.reset_name_to_default
p p.name_present? # false

kkk
```

Let's see how rails accomplishes this:

### How methods are defined

There is a class named `AttributeMethodMatcher` which is used for
storing methods. Relevant parts here:

``` ruby
class AttributeMethodMatcher
  attr_reader :prefix, :suffix, :method_missing_target

  AttributeMethodMatch = Struct.new(:target, :attr_name,
:method_name)

  def initialize(options = {})
    # other code

    @prefix, @suffix = options[:prefix] || '', options[:suffix]
|| ''
    @regex =
/^(?:#{Regexp.escape(@prefix)})(.*)(?:#{Regexp.escape(@suffix)})$/
    @method_missing_target = "#{@prefix}attribute#{@suffix}"
    @method_name = "#{prefix}%s#{suffix}"
  end

  def match(method_name)
    if @regex =~ method_name
      AttributeMethodMatch.new(method_missing_target, $1,
method_name)
    else
      nil
    end
  end

  def method_name(attr_name)
    @method_name % attr_name
  end
end
```

Every time you call either `attribute_method_affix` or
`attribute_method_prefix` or `attribute_method_suffix`, behind the scene
you create an instance of this class and store it into an array. Only
when you call `define_attribute_methods` the magic really happens. Let's
see what happens there:

``` ruby
def define_attribute_methods(attr_names)
  attr_names.each { |attr_name| define_attribute_method(attr_name)
}
end

def define_attribute_method(attr_name)
  # Remember that attribute_method_matchers contains our original matchers
  attribute_method_matchers.each do |matcher|
    method_name = matcher.method_name(attr_name)

    unless instance_method_already_implemented?(method_name)
      generate_method =
"define_method_#{matcher.method_missing_target}"

      if respond_to?(generate_method, true)
        send(generate_method, attr_name)
      else
        define_optimized_call generated_attribute_methods,
method_name, matcher.method_missing_target, attr_name.to_s
      end
    end
  end
  attribute_method_matchers_cache.clear
end

# Removes all the previously dynamically defined methods from the
class
def undefine_attribute_methods
  generated_attribute_methods.module_eval do
    instance_methods.each { |m| undef_method(m) }
  end
  attribute_method_matchers_cache.clear
end

# Returns true if the attribute methods defined have been
generated.
def generated_attribute_methods #:nodoc:
  @generated_attribute_methods ||= begin
    mod = Module.new
    include mod
    mod
  end
end
```

Pay attention to the method definition of `generated_attribute_methods`:

``` ruby
@generated_attribute_methods ||= begin
  mod = Module.new
  include mod
  mod
end
```
What's happening here is that we're creating a module, *including the
module on our class*, and then returning the module so we can keep track
of it, and *dynamically add methods to it*, which *will* be included to
our class as well. Note that we can undefine those methods at any time
by calling `undefine_attribute_methods`.

# See for yourself

There's more happening behind the scene:

[ActiveModel::AttributeMethods on Github](https://github.com/rails/rails/blob/5d0c1814ad624090620e907012e0eaf353468202/activemodel/lib/active_model/attribute_methods.rb)

* * *

[@oscardelben](http://twitter.com/oscardelben)
