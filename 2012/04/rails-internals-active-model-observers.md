# Rails Internals: Active Model Observers

Observers are those magic classes that send you notifications when your
Active Record objects are changed. Here's a common scenario that you've
probably encountered many times:

``` ruby
class UserObserver
  def after_save(user)
    # ...
  end
end
```

If that doesn't sound familiar to you, go read about observers in rails the
[documentation](http://api.rubyonrails.org/classes/ActiveRecord/Observer.html).

Still here? Let's take a journey inside Rails and see how observers are
implemented.

### Observers internals

ActiveModel defines a general `Observing` module that allows you to add
Observing like functionality to any framework. For example, here's how
you would add observing functionality to your new ORM:

``` ruby
class ORM
  include ActiveModel::Observing
end

# Calls PersonObserver.instance
ORM.observers = :person_observer
```

If you replace `ORM` with `ActiveRecord::Base`, you'll see that this is
exactly what ActiveRecord does:

``` ruby
class ActiveRecord::Base
  include ActiveModel::Observing
end

# Calls PersonObserver.instance, you usually put this in application.rb
ActiveRecord::Base.observers = :person_observer
```

Let's see how this works internally. Comments inline:

``` ruby
# Standard setters
def observers=(*values)
  observers.replace(values.flatten)
end

# ObserverArray provides some convenience methods like enable and
# disable
def observers
  @observers ||= ObserverArray.new(self)
end

def observer_instances
  @observer_instances ||= []
end

# Observers are instantiated during startup, more on this later
def instantiate_observers
  observers.each { |o| instantiate_observer(o) }
end

def add_observer(observer)
  unless observer.respond_to? :update
    raise ArgumentError, "observer needs to respond to 'update'"
  end
  observer_instances << observer
end

def notify_observers(*args)
  observer_instances.each { |observer| observer.update(*args) }
end

def count_observers
  observer_instances.size
end

protected
  def instantiate_observer(observer) #:nodoc:
    # string/symbol
    if observer.respond_to?(:to_sym)
      observer.to_s.camelize.constantize.instance
    elsif observer.respond_to?(:instance)
      observer.instance
    else
      raise ArgumentError,
        "#{observer} must be a lowercase, underscored class name (or an " +
        "instance of the class itself) responding to the instance " +
        "method. Example: Person.observers = :big_brother # calls " +
        "BigBrother.instance"
    end
  end

  # This is mostly used internally
  def inherited(subclass)
    super
    notify_observers :observed_class_inherited, subclass
  end
end

private

def notify_observers(method)
  self.class.notify_observers(method, self)
end
```

Note that your observers are expected to implement the `instance` class
method, which shuld return a shared instance of the observer class.
This method is automatically defined when you subclass `Observer` in
your observers classes. The update method is also defined when you
inherit from the `Observer` class.

### The Observer class

To add observing functionalities to your class, you inherit from
`ActiveModel::Observer`. Example:

``` ruby
class UserObserver < ActiveModel::Observer
  observe :user # Optional, can be figured out by the class name

  def after_save(user)
    ...
  end
end
```

Here's how the `Observer` class looks like:

``` ruby
class Observer
  include Singleton # defines the instance method on your observers
  extend ActiveSupport::DescendantsTracker

  class << self

    # You can pass both symbols and constants
    def observe(*models)
      models.flatten!
      models.collect! { |model| model.respond_to?(:to_sym) ? model.to_s.camelize.constantize : model }
      redefine_method(:observed_classes) { models }
    end

    # You could redefine this method to define observed_classes
    def observed_classes
      Array(observed_class)
    end

    # The class observed by default is inferred from the observer's class name:
    #   assert_equal Person, PersonObserver.observed_class
    def observed_class
      if observed_class_name = name[/(.*)Observer/, 1]
        observed_class_name.constantize
      else
        nil
      end
    end
  end

  # This is called by the instance method discussed above
  # Initialization happens only once.
  def initialize
    observed_classes.each { |klass| add_observer!(klass) }
  end

  def observed_classes #:nodoc:
    self.class.observed_classes
  end

  # Where the magic happens
  def update(observed_method, object, &block) #:nodoc:
    return unless respond_to?(observed_method)
    return if disabled_for?(object)
    send(observed_method, object, &block)
  end

  def observed_class_inherited(subclass) #:nodoc:
    self.class.observe(observed_classes + [subclass])
    add_observer!(subclass)
  end

  protected
    def add_observer!(klass) #:nodoc:
      klass.add_observer(self)
    end

    def disabled_for?(object)
      klass = object.class
      return false unless klass.respond_to?(:observers)
      klass.observers.disabled_for?(self)
    end
end
```

Nothing fancy here, but still an interesting read, and worth knowing how
this stuff works behind the scenes.

### ActiveRecord and Observers

How does ActiveRecord use observers to send notifications? Let's see.

``` ruby
module ActiveRecord
  class Observer < ActiveModel::Observer

    protected

      def observed_classes
        klasses = super
        klasses + klasses.map { |klass| klass.descendants }.flatten
      end

      def add_observer!(klass)
        super
        define_callbacks klass
      end

      def define_callbacks(klass)
        observer = self
        observer_name = observer.class.name.underscore.gsub('/', '__')

        ActiveRecord::Callbacks::CALLBACKS.each do |callback|
          next unless respond_to?(callback)
          callback_meth = :"_notify_#{observer_name}_for_#{callback}"
          unless klass.respond_to?(callback_meth)
            klass.send(:define_method, callback_meth) do |&block|
              observer.update(callback, self, &block)
            end
            klass.send(callback, callback_meth)
          end
        end
      end
  end
end
```

As it turns out, when using Observers in Rails you don't subclass from
`ActiveModel::Observer` directly, but instead use
`ActiveRecord::Observer` which sends notifications for all Active Record
callbacks. Here's a list of currently available callbacks:

``` ruby
CALLBACKS = [
      :after_initialize, :after_find, :after_touch, :before_validation,
:after_validation,
      :before_save, :around_save, :after_save, :before_create,
:around_create,
      :after_create, :before_update, :around_update, :after_update,
      :before_destroy, :around_destroy, :after_destroy, :after_commit,
:after_rollback
    ]
```

### Conclusion

Observers may seem like magic when you first encounter them, but as
you've learned today their implementation is very readable. It's also
interesting to learn that we can use observers even outside of Active
Record, for example you could easily add observers to a mongodb ORM by
subclassing `ActiveModel::Observer`.
