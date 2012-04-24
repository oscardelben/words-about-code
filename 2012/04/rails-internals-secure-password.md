# Rails Internals: Secure Password

Active Model provides an easy way to add password authentication to your
models without relying on third party tools. Here's a basic example:

    require 'rubygems'
    require 'active_model'
    require 'bcrypt-ruby'

    class User
      include ActiveModel::SecurePassword
      include ActiveModel::Validations

      attr_accessor :password_digest # This would normally be a database
    column

      has_secure_password
    end

    user = User.new
    user.password = 'test'
    user.password_confirmation = 'test'
    p user.authenticate 'sdas' # => false
    p user.authenticate 'test' # => user

SecurePassword's implementation is almost trivial. It uses bcrypt behind
the scenes and the whole implementation fits inside a single, small
file:

    module ActiveModel
      module SecurePassword
        extend ActiveSupport::Concern

        module ClassMethods

          def has_secure_password
            # Load bcrypt-ruby only when has_secure_password is used.
            # This is to avoid ActiveModel (and by extension the entire framework) being dependent on a binary library.
            gem 'bcrypt-ruby', '~> 3.0.0'
            require 'bcrypt'

            attr_reader :password

            validates_confirmation_of :password
            validates_presence_of     :password_digest

            include InstanceMethodsOnActivation

            if respond_to?(:attributes_protected_by_default)
              def self.attributes_protected_by_default
                super + ['password_digest']
              end
            end
          end
        end

        module InstanceMethodsOnActivation
          # Returns self if the password is correct, otherwise false.
          def authenticate(unencrypted_password)
            BCrypt::Password.new(password_digest) == unencrypted_password && self
          end

          # Encrypts the password into the password_digest attribute.
          def password=(unencrypted_password)
            unless unencrypted_password.blank?
              @password = unencrypted_password
              self.password_digest = BCrypt::Password.create(unencrypted_password)
            end
          end
        end
      end
    end

Notice how `password_digest` is added to the attributes protected by
default, and that a blank password is not allowed. You can overwrite
`authenticate` and `password=` to provide custom behavior, but the
implementation is so simple that your best bet at this time is to roll
your own authentication system or use one of the third party libraries
available.
