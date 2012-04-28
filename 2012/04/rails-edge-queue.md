# Rails Edge: Queue

Rails just added support for an unified api for Queue. The goal is to
provide a common interface that extenal libraries can use without having to
worry about implementation details.

Here's how the public api looks so far:

``` ruby
Rails.queue.push job
```

Where job is an object that should respond to `run`.

You can create your own Queue and add it to Rails via a config setting:

``` ruby
config.queue = MyQueue
```

Here are two examples of Queue implementation from the Rails source:

``` ruby
class TestQueue < ::Queue
  def jobs
    @que.dup
  end

  def drain
    Thread.new { pop.run until empty? }.join
  end
end

class ThreadedConsumer
  def self.start(queue)
    new(queue).start
  end

  def initialize(queue)
    @queue = queue
  end

  def start
    @thread = Thread.new do
      while job = @queue.pop
        begin
          job.run
        rescue Exception => e
          Rails.logger.error "Job Error:
#{e.message}\n#{e.backtrace.join("\n")}"
        end
      end
    end
    self
  end

  def shutdown
    @queue.push nil
    @thread.join
  end
end
```

