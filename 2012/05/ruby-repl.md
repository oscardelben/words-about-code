# A Simple REPL in Ruby

This is a simple, single line Ruby Repl which shows the basics of how
irb works. It's super lightweight and minimalist, and probably not very
usable as it is.

```ruby
should_exit = false
_ = nil

print "> "

while !should_exit && line = gets.chomp
  if line.chomp.strip == 'exit'
    should_exit = true
    next
  end

  begin
    _ = eval(line)
    p _
  rescue Exception => e
    puts e.message
  ensure
    print "> "
  end
end
```

From here, adding multiline support requires some work. Most likely, you
want to save input in a buffer and use a scanner to detect when input is
finished. If I have time I'll make one just for fun.
