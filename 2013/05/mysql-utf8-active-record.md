# Problems With Mysql UTF-8 Charset and Active Record

Many people (including myself until recently) are unaware of the fact that [mysql utf-8 encoding can only handle upto 3 bytes characters](http://dev.mysql.com/doc/refman/5.5/en/charset-unicode-utf8mb4.html).

If you accept unicode characters inside your application this may cause problems as mysql will truncate everything including and after the 4 byte character.

```
message = Message.new
message.body = "a simple \ud83d\udcaa test"
message.save
message.body # => "a simple \ud83d\udcaa test"
message.reload.body # => "a simple "
```

As you can see the message gets truncated. 

The fix depends on wherever you want to patch ActiveRecord or do a conversion only on the fields you care about. Here's how you would overwrite a specific field:

```ruby
def body=(value)
  write_attribute :body, value.gsub(/[\u{10000}-\u{10FFFF}]/, "something")
end
```

This solution is pretty fast (can convert a 1 mil string in ~20ms) and non invasive. If you want to monkey patch every field you'll have to hook into ActiveRecord write_attribute directly with similar outcomes.

Note that this is only a temporary hack, you should upgrade to mysql 5.5+ and use the utf8mb4 character set which fully supports 4 byte characters.
