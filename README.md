# xchan.rb

xchan.rb is a library for sending Ruby objects
between Ruby processes who have a parent&lt;-&gt;child relationship. The
implementation uses a <code><a href=https://rubydoc.info/stdlib/socket/UNIXSocket.pair>UNIXSocket</a></code>
and the serialization format of your choice - the default is [`Marshal`](https://www.rubydoc.info/stdlib/core/Marshal).


## Examples

**Serializers**

When a channel is written to and read from, a Ruby object is serialized (on write)
and deserialized (on read). The form of serialization used can be customized by
the first argument given to `xchan()`. For instance any of the following could be
used: `xchan(:marshal)`, `xchan(:json)`, or `xchan(:yaml)`. The example uses
[`Marshal`](https://www.rubydoc.info/stdlib/core/Marshal):

```ruby
require "xchan"

##
# This channel uses Marshal to serialize objects.
ch = xchan
ch.send(msg: "serialized by Marshal")
Process.wait fork { print "Received message: ", ch.recv[:msg], "\n" }
ch.close

##
# This channel also uses Marshal to serialize objects.
ch = xchan(:marshal)
ch.send(msg: "serialized by Marshal")
Process.wait fork { print "Received message: ", ch.recv[:msg], "\n" }
ch.close

##
# Received message: serialized by Marshal
# Received message: serialized by Marshal
```

**Blocking read**

The following example demonstrates how to send a Ruby object from a parent process
to a child process. `ch.recv` performs a blocking read until an object is sent
to the channel. The example sends a random Integer:

```ruby
require "xchan"

ch = xchan
pid = fork do
  print "Received a random number (child process): ", ch.recv, "\n"
end
print "Send a random number (from parent process)", "\n"
ch.send(rand(21))
Process.wait(pid)
ch.close

##
# Send a random number (from parent process)
# Received random number (child process): XX
```

**Queue messages**

The following example demonstrates how a channel can queue messages that
can later be read one by one. The order in which the messages
are read from the channel follows the
[First In, First out (FIFO)](https://en.wikipedia.org/wiki/FIFO_(computing_and_electronics))
methodology. In other words the example will read messages in the
order they were sent: 1 first, then 2, and finally 3:

```ruby
require "xchan"

ch = xchan
Process.wait fork {
  print "Queue messages (from child process)", "\n"
  ch.send(1)
  ch.send(2)
  ch.send(3)
}
3.times { print "Received (parent process): ", ch.recv, "\n" }
ch.close

##
# Queue messages (from child process)
# Received (parent process): 1
# Received (parent process): 2
# Received (parent process): 3
```

**Count objects**

The following example demonstrates how the `#size` method can be
used to count how many objects are waiting to be read from a channel:

```ruby
require "xchan"

ch = xchan
3.times do |i|
  Process.wait fork { ch.send([i]) }
end
3.times do
  print "channel size: ", ch.size, "\n"
  print "read: ", ch.recv, "\n"
end
print "channel size: ", ch.size, "\n"
ch.close

##
# channel size: 3
# read: [0]
# channel size: 2
# read: [1]
# channel size: 1
# read: [2]
# channel size: 0
```

**Bytes in, bytes out**

The following example demonstrates how the number of bytes sent to and received
from a channel can be tracked using the `#bytes_sent` and `#bytes_received` methods:

```ruby
require "xchan"

ch = xchan
Process.wait fork { ch.send %w[0x1eef] }
print "Bytes sent: ", ch.bytes_sent, "\n"
Process.wait fork { ch.recv }
print "Bytes received: ", ch.bytes_received, "\n"
ch.close

##
# Bytes sent: 18
# Bytes received: 18
```

## Advanced examples

**Parallel map**

The following example demonstrates a method by the name `p_map` -
implemented in 10 LOC - that runs a map operation in parallel.
There is a slight overhead - less than a tenth of a second - for
an operation that would otherwise take 6 seconds to execute sequentially:

```ruby
require "xchan"

def p_map(enum)
  ch = xchan
  enum.map
      .with_index { |e, i| fork { ch.send [yield(e), i] } }
      .each { Process.wait(_1) }
  enum.map { ch.recv }
      .tap { ch.close }
      .sort_by(&:pop)
      .map(&:pop)
end

t = Time.now
print p_map([3, 2, 1]) { |e| sleep(e).then { e * 2 } }, "\n"
print format("Duration: %.2f", Time.now - t), "\n"

##
# == Output
# [6, 4, 2]
# Duration: 3.01
```


**Consume contents**

**#to_a**

The following example demonstrates how the `#to_a` method can be
used to consume and return the contents of a channel as an Array:

```ruby
require "xchan"

ch = xchan
1.upto(5) { ch.send(_1) }
print "Read from populated channel ", ch.to_a, "\n"
print "Read from empty channel ", " " * 4, ch.to_a, "\n"
ch.close

##
# Read from populated channel [1, 2, 3, 4, 5]
# Read from empty channel     []
```

**Splat**

The following example demonstrates how the splat operator can be
used to consume and forward the contents of a channel as arguments
to a method:

```ruby
def sum(a, b, c, d)
  [a, b, c, d].sum
end

ch = xchan
1.upto(4) { ch.send(_1) }
print "Sum: ", sum(*ch), "\n"
ch.close

##
# Sum: 10
```

## Resources

* [Source code (GitHub)](https://github.com/0x1eef/xchan.rb)
* [Documentation](https://0x1eef.github.io/x/xchan.rb)

## Install

xchan.rb is available as a RubyGem:

    gem install xchan.rb

## <a id="license"> License </a>

The MIT license, see [LICENSE.txt](./LICENSE.txt) for details.
