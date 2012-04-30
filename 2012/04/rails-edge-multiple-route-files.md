# Rails Edge: Multiple Route Files

Rails edge just got a great addition, multiple route files:

``` ruby
# config/routes.rb
draw :admin

# config/routes/admin.rb
namespace :admin do
  resources :posts
end
```

This will come handy for breaking down complex route files in large
apps.
