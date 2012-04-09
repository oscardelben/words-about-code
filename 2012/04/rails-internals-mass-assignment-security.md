# Rails Internals: Mass Assignment Security

**IMPORTANT** This article is still a draft. It is not completed and
the information it contains may be wrong. Please be patient.

Mass assigment security has recently been the focus of a debate in the
Ruby community after a fellow hacker demostrated that lots of apps
suffer from bad defaults when protecting model attributes.

Consider this example:

``` ruby
class User < ActiveRecord::Base
  attr_accessor :name, :role, :updated_at

  attr_protected :role
end
```

In this code we're protecting from a malicious assignment of the
`role` attribute, but notice how we also expose `updated_at` which is
probably not what we intended. Now, in the real world attributes will be
generated automatically and so you'll be more vulnerable to these kind
of mistakes.

### attr_accessible

`attr_accessible` is the obvious solution to this. It allows to specify
exactly which attributes you want to expose to mass assignment.

``` ruby
class User < ActiveRecord::Base
  attr_accessor :name, :role, :updated_at

  attr_accessible :name
  # We can also pass a role option!
  attr_accessible :name, :role, :updated_at, :as => 'admin'
end
```

Match better. Now let's see how these methods are defined internally.

## ActiveModel::MassAssignmentSecurity

We'll start our journey inside `ActiveModel::MassAssignmentSecurity`,
which provides a general interfacer

``` ruby
def attr_protected(*args)
  options = args.extract_options!
  role = options[:as] || :default

  self._protected_attributes = protected_attributes_configs.dup

  Array(role).each do |name|
    self._protected_attributes[name] = self.protected_attributes(name) + args
  end

  self._active_authorizer = self._protected_attributes
end

def attr_accessible(*args)
  options = args.extract_options!
  role = options[:as] || :default

  self._accessible_attributes = accessible_attributes_configs.dup

  Array(role).each do |name|
    self._accessible_attributes[name] = self.accessible_attributes(name) + args
  end

  self._active_authorizer = self._accessible_attributes
end
```

`attr_protected` and `attr_accessible` are very similar. They both
accept an optional role[s] parameter for enhanced security.

It's interesting to see how we can use `Array(elt)` to turn an element
to an array, unless it's already an array:

``` ruby
A1.9.3p125 :001 > Array(:foo)
 => [:foo]
1.9.3p125 :002 > Array([:foo])
 => [:foo]
```

Let's move on:

``` ruby
def protected_attributes(role = :default)
  protected_attributes_configs[role]
end

def accessible_attributes(role = :default)
  accessible_attributes_configs[role]
end

def active_authorizers
  self._active_authorizer ||= protected_attributes_configs
end
alias active_authorizer active_authorizers

def attributes_protected_by_default
  []
end

def mass_assignment_sanitizer=(value)
  self._mass_assignment_sanitizer = if value.is_a?(Symbol)
    const_get(:"#{value.to_s.camelize}Sanitizer").new(self)
  else
    value
  end
end
```

Here we see that we can use `mass_assignment_sanitizer=` to define our
own sanitizer, but perhaps more interesting there's a method called
`attributes_protected_by_default`, which is empty here, but you could
overwrite it to return something else (useful in subclassess, I
guess).

I'll show you what a **sanitizer** looks like later. Now we'll introduce
another concept, **authorizer**, which is either a `WhiteList`, or
`Blacklist` class those responsibility is to tell wherever an attribute
had to be denied, or not. Here's where we instantiate those classes:
show

``` ruby

def protected_attributes_configs
  self._protected_attributes ||= begin
    Hash.new { |h,k| h[k] = BlackList.new(attributes_protected_by_default) }
  end
end

def accessible_attributes_configs
  self._accessible_attributes ||= begin
    Hash.new { |h,k| h[k] = WhiteList.new }
  end
end
```

That `Hash.new` syntax is basically defining a *default* value for the
hash.

Here's how everything fits together:

``` ruby
def sanitize_for_mass_assignment(attributes, role = nil)
  _mass_assignment_sanitizer.sanitize(attributes, mass_assignment_authorizer(role))
end

def mass_assignment_authorizer(role)
  self.class.active_authorizer[role || :default]
end
```

`sanitize_for_mass_assignment` is what you would usually call from your
models, and it would just call your sanitizer of choice (default `logger`,
but you can pass `strict` to raise an error instead), with your
authorizer.

For completeness, I'm pasting the content of the sanitizer and authorizer classes.

``` ruby
module ActiveModel
  module MassAssignmentSecurity
    class Sanitizer
      # Returns all attributes not denied by the authorizer.
      def sanitize(attributes, authorizer)
        rejected = []
        sanitized_attributes = attributes.reject do |key, value|
          rejected << key if authorizer.deny?(key)
        end
        process_removed_attributes(rejected) unless rejected.empty?
        sanitized_attributes
      end

    protected

      def process_removed_attributes(attrs)
        raise NotImplementedError, "#process_removed_attributes(attrs) suppose to be overwritten"
      end
    end

    class LoggerSanitizer < Sanitizer
      def initialize(target)
        @target = target
        super()
      end

      def logger
        @target.logger
      end

      def logger?
        @target.respond_to?(:logger) && @target.logger
      end

      def process_removed_attributes(attrs)
        logger.warn "Can't mass-assign protected attributes: #{attrs.join(', ')}" if logger?
      end
    end

    class StrictSanitizer < Sanitizer
      def initialize(target = nil)
        super()
      end

      def process_removed_attributes(attrs)
        return if (attrs - insensitive_attributes).empty?
        raise ActiveModel::MassAssignmentSecurity::Error.new(attrs)
      end

      def insensitive_attributes
        ['id']
      end
    end

    class Error < StandardError
      def initialize(attrs)
        super("Can't mass-assign protected attributes: #{attrs.join(', ')}")
      end
    end
  end
end
```
``` ruby
require 'set'

module ActiveModel
  module MassAssignmentSecurity
    class PermissionSet < Set

      def +(values)
        super(values.map(&:to_s))
      end

      def include?(key)
        super(remove_multiparameter_id(key))
      end

      def deny?(key)
        raise NotImplementedError, "#deny?(key) supposed to be overwritten"
      end

    protected

      def remove_multiparameter_id(key)
        key.to_s.gsub(/\(.+/, '')
      end
    end

    class WhiteList < PermissionSet

      def deny?(key)
        !include?(key)
      end
    end

    class BlackList < PermissionSet

      def deny?(key)
        include?(key)
      end
    end
  end
end
```

### How ActiveRecord Uses Mass Assignment to protect attributes

ActiveRecord includes the Mass Assignment module to provide
`attr_accessible` and `attr_protected` methods for your models.

Beside that, it redefines `attributes_protected_by_default` to protect
the primary_key and Inheritance_column from mass assignment:

```ruby
def attributes_protected_by_default
  default = [ primary_key, inheritance_column ]
  default << 'id' unless primary_key.eql? 'id'
  default
end
```

Also worth our attention is that the `attributes=` method of
ActiveRecord calls `assign_attributes` which you could also call
explicitly to skip mass assignment validation:

``` ruby
def attributes=(new_attributes)
  return unless new_attributes.is_a?(Hash)

  assign_attributes(new_attributes)
end

def assign_attributes(new_attributes, options = {})
  # Other uninteresting code

  unless options[:without_protection]
    attributes = sanitize_for_mass_assignment(attributes,
mass_assignment_role)
  end
  # other code
end
```

Here's some examples from the rails documentation:

```
#   user = User.new
#   user.assign_attributes({ :name => 'Josh', :is_admin => true }, :without_protection => true)
#   user.name       # => "Josh"
#   user.is_admin?  # => true
```

### Summary

By studing how mass assignment works, we've learned how we can add
attributes protection to any class, specify default attributes
protection (useful if you're building an external library), and skip
validation if you want to (admin panels anyone). You also learned that
you can add role attributes for enhanced protection.
