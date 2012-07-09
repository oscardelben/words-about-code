# Rails Edge: Custom Flash Types

Rails just got some syntactic sugar for dealing with flash messages in
your controllers and views.

You can now declare the following in your controllers:

```ruby
class ApplicationController
  add_flash_types :error, :warning
end
```

And then you'll be able to use `<%= error %>` in your views and
`redirect_to .., error: "message"` in your controllers.

Commit message: [Added support add_flash_types](https://github.com/rails/rails/commit/238a4253bf229377b686bfcecc63dda2b59cff8f)
