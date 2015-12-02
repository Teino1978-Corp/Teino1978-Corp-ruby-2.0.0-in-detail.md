# Ruby 2.0.0 in detail #

### Keyword arguments ###

```ruby
def wrap(string, before: "<", after: ">")
  "#{before}#{string}#{after}" # no need to retrieve options from a hash
end

# optional
wrap("foo")                                  #=> "<foo>"
# one or the other
wrap("foo", before: "#<")                    #=> "#<foo>"
wrap("foo", after: "]")                      #=> "<foo]"
# order not important
wrap("foo", after: "]", before: "[")         #=> "[foo]"
```

One of the nice things about this compared to the hash-as-fake-keyword-args
is that it errors when you make a mistake.

```ruby
begin
  wrap("foo", befoer: "#<")
rescue ArgumentError => e
  e.message                                  #=> "unknown keyword: befoer"
end
```

You can use double splat to capture all keyword arguments, just like a single
splat to capture all regular arguments. You can also use the double splat to
unpack a hash to keyword arguments.

```ruby
# arguments
def capture(**opts)
  opts
end
capture(foo: "bar")                          #=> {:foo=>"bar"}

# keys must be symbols
opts = {:before => "(", :after => ")"}
wrap("foo", **opts)                          #=> "(foo)"
```

The old hash style syntax is still accepted for keyword arguments, so you can
update your method definitions without having to fix all the callers.

```ruby
wrap("foo", :before => "{", :after => "}")   #=> "{foo}"
```

If you're writing a library that needs to be compatible across ruby 2.0 and 1.9
the old hash-as-fake-keyword-args trick still works, and can be called as if it
was using keyword arguments

```ruby
def wrap(string, opts={})
  before = opts[:before] || "<"
  after = opts[:after] || ">"
  "#{before}#{string}#{after}"
end

wrap("foo", before: "[", after: "]")         #=> "[foo]"
```

### %i and %I symbol array literal ###

```ruby
%i{an array of symbols}   #=> [:an, :array, :of, :symbols]
```

and with interpolation

```ruby
%I{#{1 + 1}x #{2 + 2}x}   #=> [:"2x", :"4x"]
```

### Refinements ###

Refinements is a neat idea, but the original implementation came with a weird
edge cases and possible performance penalties, so what we get with Ruby 2.0.0
is a rather scaled back, and slightly less useful version of the original.

You create a refinement to a class, name spaced inside a module

```ruby
module NumberQuery
  refine String do
    def number?
      match(/\A[1-9][0-9]*\z/) ? true : false
    end
  end
end
```

this refinement isn't visible by default

```ruby
"123".respond_to?(:number)   #=> false
```

once you declare that you are 'using' the module with the refinement, then it
becomes visible.

```ruby
using NumberQuery
"123".number?                #=> true
```

however `using` is only available at the top level, and only applies to lines
after it in the same source file it's declared. You also get a warning that
refinements are experimental if you're running with warnings enabled.

### Module#prepend ###

Module gains `#prepend` as a compliment to `#include`, it works just like
include, but it inserts the module in to the inheritance chain as if it were a
subclass rather than a superclass.

```ruby
Object
superclass
included module
class
prepended module
```

This takes over from Rails' `#alias_method_chain` or the trick of aliasing a
method to a new name before redefining it and calling the original.

```ruby
class Foo
  def do_stuff
    puts "doing stuff"
  end
end

module Wrapper
  def do_stuff
    puts "before stuff"
    super
    puts "after stuff"
  end
end

class Foo
  prepend Wrapper
end

Foo.new.do_stuff
```

outputs:

    before stuff
    doing stuff
    after stuff

there's also `::prepended` and `::prepend_features` method that work like
`::included` and `::append_features`

### Unbound methods from a module can be bound to any object ###

This one might sound like gibberish, or some minor change to something you'll
never use, but it's actually a really great new feature.

You can get ahold of a method object from any class or module, with
instance_method

```ruby
Object.instance_method(:to_s)   #=> #<UnboundMethod: Object(Kernel)#to_s>
```

However, this method isn't bound to anything, it has no 'self', and can't be
called. To call it you have to bind the method to an object, but methods have
to be bound to an object of the same class as the one the method was taken from.

But now we can take a method from a module and bind it to any object

```ruby
module Bar
  def bar
    "bar"
  end
  def baz
    "baz"
  end
end

Bar.instance_method(:bar).bind(Object.new).call   #=> "bar"
```

This means define_method also accepts unbound methods for modules, which will
let us implement a selective include

```ruby
module Kernel
  def from(mod, include: [])
    raise TypeError, "argument must be a module" unless Module === mod
    include.each do |name, original|
      define_method(name, mod.instance_method(original || name))
    end
  end
end

class Foo
  from Bar, include: {:qux => :bar}
end

f = Foo.new
f.qux                 #=> "bar
f.respond_to?(:baz)   #=> false
```

### const_get understands namespaces ###

```ruby
class Foo
  module Bar
    Baz = 1
  end
end

Object.const_get("Foo::Bar::Baz")   #=> 1
```

### #to_h as standard for 'convert to hash' ###

Hash, along with ENV, nil, Struct, and OpenStruct all get a `#to_h` method which
returns a hash

```ruby
{:foo => "bar"}.to_h               #=> {:foo=>"bar"}
nil.to_h                           #=> {}
Struct.new(:foo).new("bar").to_h   #=> {:foo=>"bar"}
require "ostruct"
open = OpenStruct.new
open.foo = "bar"
open.to_h                          #=> {:foo=>"bar"}
```

There is also a Hash() method, like Array(), that delegates to `#to_h`

```ruby
Hash({:foo => "bar"})              #=> {:foo=>"bar"}
Hash(nil)                          #=> {}
```

### Array#bsearch and Range#bsearch ###

Array and Range get a binary search method with `#bsearch`. This has two modes of
working, find-minimum and find-any. Both modes take a block, and the array will
have to be sorted with regards to this block.

In find-minimum mode it will return the first element greater than or equal to
a chosen value. To use this mode you supply a block that returns true when the
supplied element is greater than or equal to the chosen value, and false
otherwise.

```ruby
array = [2, 4, 8, 16, 32]
array.bsearch {|x| x >= 4}       #=> 4
array.bsearch {|x| x >= 7}       #=> 8
array.bsearch {|x| x >= 9}       #=> 16
array.bsearch {|x| x >= 0}       #=> 2
array.bsearch {|x| x >= 33}      #=> nil
```

In find any mode you need to supply a block that returns a positive number if
the supplied element is less than your chosen value, a negative number if it is
greater, and 0 if it is the chosen value.

```ruby
array.bsearch {|x| 4 <=> x}      #=> 4
array.bsearch {|x| 7 <=> x}      #=> nil
```

Your block can return 0 for a range of values, in which case any of them may be
chosen.

```ruby
array = [0, 4, 7, 10, 12]
array.map {|x| 1 - x / 4 }       #=> [1, 0, 0, -1, -2]
array.bsearch {|x| 1 - x / 4 }   #=> 4 or 7
```

### Enumerable#lazy ###

`#lazy` called on any Enumerable (Array, Hash, File, Range, etc) will return a
lazy enumerator, that doesn't perform any calculations till it is forced to. In
addition, elements of the enumerable will be sent though the whole chain one by
one, rather than evaluating the entire enumerable at each step. In some cases
this will result in less work.

You can deal with infinite collections

```ruby
[1,2,3].lazy.cycle.map {|x| x * 10}.take(5).to_a   #=> [10, 20, 30, 10, 20]
```

Or avoid consuming large resources, when you know you'll only need a little

```ruby
File.open(__FILE__).lazy.each.map(&:chomp).reject(&:empty?).take(3).force
```

This example only reads as many lines as it takes to find 3 that aren't empty.

`#lazy` does incur a performance penalty, so you'll need to make sure you're only
using it when it makes sense.

There's also a Enumerator::Lazy class to create your own lazy enumerators.
The example above could be written as:

```ruby
def populated_lines(file, &block)
  Enumerator::Lazy.new(file) do |yielder, line|
    string = line.chomp
    yielder << string unless string.empty?
  end.each(&block) # evals block, or returns enum if nil, like stdlib
end

populated_lines(File.open(__FILE__)).take(3).force
```

### Lazy Enumerator#size and Range#size ###

`Enumerator#size` will return the size of an enumerator, without evaluating the
whole Enumerator. Range benefits from this too.

```ruby
array = [1,2,3,4]
array.cycle(4).size    #=> 16
array.cycle.size       #=> Infinity
# nil is returned if the size can't be calculated
array.find.size        #=> nil
# Range too
(1..10).size           #=> 10
```

To enable this in Enumerators returned from your own code, Enumerator.new now
accepts an argument to calculate the size.

This can be a value

```ruby
  enum = Enumerator.new(3) do |yielder|
    yielder << "a"
    yielder << "b"
    yielder << "c"
  end

  enum.size              #=> 3
```

or an object responding to `#call`

```ruby
def square_times(num, &block)
  Enumerator.new(-> {num ** 2}) do |yielder|
    (num ** 2).times {|i| yielder << i}
  end.each(&block)
end

square_times(6).size     #=> 36
```

`#to_enum` (and it's alias `#enum_for`) also now take a block to calculate the
size, so the above could be written like so:

```ruby
def square_times(num)
  return to_enum(:square_times) {num ** 2} unless block_given?
  (num ** 2).times do |i|
    yield i
  end
end

square_times(6).size     #=> 36
```

### Rubygems Gemfile support ###

Rubygems can now use your Gemfile (or Isolate, or gem.deps.rb) to install gems
and load activation information.

You can install the gems listed in your Gemfile (and their dependancies) by
specifying the `--file` (or `-g`) option. This only uses the Gemfile, not
Gemfile.lock, so only versions specified in the Gemfile will be respected.

```sh
gem install --file Gemfile
```

To load activation information (the version of the gems to use) specify the
`RUBYGEMS_GEMDEPS` environment variable. The value for this should be the path
to your Gemfile, but you can use `-` to have Rubygems auto-detect this.

Like install, this only uses the Gemfile, ignoring Gemfile.lock, so it's less
strict than Bundler. It also only activates the specified versions, you'll
still need to require the gems in your code.

It does have the advantage that as it's built in, there's no need for something
similar to `bundle exec` or Bundlers binstubs, just run your app like normal.

```sh
export RUBYGEMS_GEMDEPS=-
# start your app
```

There is only basic support for the Gemfile format, it doesn't understand the
`gemspec` declaration for example, but it's a handy feature that will hopefully
evolve and become a bit more robust, and it's very useful if Bundler isn't
working out for you.

A rough approximation of `bundle install --path vendor/bundle` can be had with

```sh
gem install --file Gemfile --install-dir vendor/gem

export GEM_HOME=vendor/gem
export RUBYGEMS_GEMDEPS=-
# start your app
```

### RDoc markdown support ###

RDoc now understands markdown, to run rdoc with markdown formatting set the 
markup option

```sh
rdoc --markup markdown
```

This can be saved in your project with a .doc_options file so you don't need to repeat it every time

```sh
rdoc --markup markdown --write-options
```

### Use warn like puts ###

warn now works just like puts, taking multiple arguments, or an array, and
outputting them, but on stderr rather than stdout

```ruby
warn "foo", "bar"
warn ["foo", "bar"]
```

### Logger compatible syslog interface ###

Now it's super easy to switch between logging to a file descriptor and to
syslog, no more picking one and then writing to that interface. You can even
log to stdout in development, and syslog in production, without having to
update all your logging calls.

```ruby
if ENV["RACK_ENV"] == "production"
  require "syslog"
  logger = Syslog::Logger.new("my_app")
else
  require "logger"
  logger = Logger.new(STDOUT)
end

logger.debug("about to do stuff")
begin
  do_stuff
rescue => e
  logger.error(e.message)
end
logger.info("stuff done")
```

### TracePoint ###

TracePoint is a new object oriented version of the old `Kernel#set_trace_func`.
It allows you to trace the execution of your Ruby code, this can be really
handy when debugging code that isn't terribly straight forward.

```ruby
# set up our tracer, but don't enable it yet
events = %i{call return b_call b_return raise}
trace = TracePoint.new(*events) do |tp|
  p [tp.event, tp.event == :raise ? tp.raised_exception.class : tp.method_id, tp.path, tp.lineno]
end

def twice
  result = []
  result << yield
  result << yield
  result
end

def check_idempotence(&block)
  a, b = twice(&block)
  raise "expected #{a} to equal #{b}" unless a == b
  true
end

trace.enable
a = 1
begin
  check_idempotence {a += 1}
rescue
end
trace.disable
```

outputs:

    [:call, :check_idempotence, "/Users/mat/Dropbox/ruby-2.0.0.rb", 841]
    [:call, :twice, "/Users/mat/Dropbox/ruby-2.0.0.rb", 834]
    [:b_call, nil, "/Users/mat/Dropbox/ruby-2.0.0.rb", 850]
    [:b_return, nil, "/Users/mat/Dropbox/ruby-2.0.0.rb", 850]
    [:b_call, nil, "/Users/mat/Dropbox/ruby-2.0.0.rb", 850]
    [:b_return, nil, "/Users/mat/Dropbox/ruby-2.0.0.rb", 850]
    [:return, :twice, "/Users/mat/Dropbox/ruby-2.0.0.rb", 839]
    [:raise, #<RuntimeError: expected 2 to equal 3>, "/Users/mat/Dropbox/ruby-2.0.0.rb", 843]
    [:return, :check_idempotence, "/Users/mat/Dropbox/ruby-2.0.0.rb", 843]

### Asynchronous Thread interrupt handling ###

Ruby threads can be killed or have an exception raised in them by another.
This isn't a terribly safe feature as the killing thread doesn't know what the
thread being killed is doing, you could end up killing a thread in the middle
of some important resource allocation or deallocation. We now have a feature
to deal with this more safely.

As an example, the stdlib timeout library works by spawning a second thread,
which waits for the specified amount of time, and then raises and exception in
the original thread.
Say we had a connection pool library that could cope fine with you failing to
check back in a connection, but would fail if checking out or checking in a
connection was interrupted. You're writing a method to get a connection, make
a request on it, then return it, and you suspect that users of this method may
wrap it in a timeout.

```ruby
def request(details)
  result = nil
  # Block will defer given exceptions if they are raised in this thread by
  # another thread till the end of the block. Exceptions are not rescured or
  # ignored, but handled later.
  Thread.handle_interrupt(Timeout::ExitException => :never) do
    # no danger of timeout interrupting checkout
    connection = connection_pool.checkout
    # if checkout took too long, handle the interrupt immediately, effectively
    # raising the pending exception here
    if Thread.pending_interrupt?
      Thread.handle_interrupt(Timeout::ExitException => :immediate)
    end
    # allow interrupts during IO (or C extension call)
    Thread.handle_interrupt(Timeout::ExitException => :on_blocking) do
      result = connection.request(details)
    end
    # no danger of timeout interrupting checkin
    connection_pool.checkin(connection)
  end
end
```

This method can safely be wrapped in a timeout, and the connection will always
be completely checked out, and if it completes the request, will always be
completes checked in. If the timeout happens during checkin, it won't interrupt
the checkin, but it will still be raised at the end of the method.

This is a slightly contrived example, but it covers the main points of this
great new feature.

### Garbage collection improvements ###

There are a few improvements to the garbage collector in Ruby 2.0, the main one
making Ruby play nicer with Copy-on-Write. This means applications that fork
multiple processes, like a Rails app running on Unicorn, will use less memory.

The GC::Profiler class also gets a `::raw_data` method, to return the raw data
of the profile as an array of hashes, rather than a string, making it easier to
log this data with say, statsd.

```ruby
GC::Profiler.enable # turn on the profiler
GC.start # force a GC run, so there will be some stats
GC::Profiler.raw_data
#=> [{:GC_TIME=>0.0012150000000000008, :GC_INVOKE_TIME=>0.036716,
#   :HEAP_USE_SIZE=>435920, :HEAP_TOTAL_SIZE=>700040,
#   :HEAP_TOTAL_OBJECTS=>17501, :GC_IS_MARKED=>0}]
```

### ObjectSpace.reachable_objects_from ###

This method returns all the objects directly reachable from the given object.

```ruby
require "objspace"

Response = Struct.new(:code, :header, :body)
res = Response.new(200, {"Content-Length" => "12"}, "Hello world!")

ObjectSpace.reachable_objects_from(res)
#=> [Response, {"Content-Length"=>"12"}, "Hello world!"]
```

You can combine this with ObjectSpace.memsize_of to get an idea of the memory
size of an object and all the objects it references; very handy for debugging
memory leaks

```ruby
def memsize_of_all_reachable_objects_from(obj)
  memsize = 0
  seen = {}.tap(&:compare_by_identity)
  to_do = [obj]
  while obj = to_do.shift
    ObjectSpace.reachable_objects_from(obj).each do |o|
      next if seen.key?(o) || Module === o
      seen[o] = true
      memsize += ObjectSpace.memsize_of(o)
      to_do << o
    end
  end
  memsize
end

memsize_of_all_reachable_objects_from(res)   #=> 192
```

### Optimised backtrace ###

Backtrace strings are now only created on demand, from a light weight
collection of object, rather than with each exception. You can get ahold of
these objects with caller_locations or `Thread#backtrace_locations`. Curiously
they aren't available from an Exception.

```ruby
def foo
  bar
end

def bar
  caller_locations
end

locations = foo
locations.map(&:label)   #=> ["foo", "<main>"]
locations.first.class    #=> Thread::Backtrace::Location
```

caller also now accepts a limit as well as an offset, or a range

```ruby
def bar
  caller(2, 1)
end
foo   #=> ["/Users/mat/Dropbox/ruby-2.0.0.rb:361:in `<main>'"]

def bar
  caller(2..2)
end
foo   #=> ["/Users/mat/Dropbox/ruby-2.0.0.rb:368:in `<main>'"]
```

### Zlib streaming support ###

Zlib gets support for streaming decompression and improved support for streaming
compression.

When decompressing files it might be reasonable to buffer the whole compressed
file in to memory, but the uncompressed data may be many hundreds of MB.
`Inflate#inflate` now accepts a block that will get chunks of data as the
file in uncompressed, this can process the uncompressed data without needing to
buffer all of it in memory at once.

```ruby
require "zlib"

inflater = Zlib::Inflate.new(Zlib::MAX_WBITS + 32)
File.open("app.log", "w") do |out|
  inflater.inflate(File.read("app.log.gz")) {|chunk| out << chunk}
end
inflater.close
```

In a similar fashion `Deflate#deflate` also accepts a block, as you feed
data in to `#deflate` the block will only be called when enough data has
accumulated to warrant efficient compression.

```ruby
deflater = Zlib::Deflate.new
File.open("app.log.gz", "w") do |out|
  File.foreach("app.log", 4 * 1024) do |chunk|
    deflater.deflate(chunk) {|part| out << part}
  end
  deflater.finish {|part| out << part}
end
```

### Multithreaded Zlib processing ###

Zlib no longer holds the Global Interpreter Lock while compressing/uncompressing
data, allowing parallel processing on gzip, zlib and deflate streams.
This also means your application can continue to respond while Zlib works in the
background.

```ruby
require "zlib"

# processes 4 files in parallel, using 4 cores if required
threads = %W{a.txt.gz b.txt.gz c.txt.gz d.txt.gz}.map do |path|
  Thread.new do
    inflater = Zlib::Inflate.new(Zlib::MAX_WBITS + 32)
    File.open(path.chomp(File.extname(path)), "w") do |out|
      inflater.inflate(File.read(path)) {|chunk| out << chunk}
    end
    inflater.close
  end
end
# do other stuff here while Zlib works in other threads
threads.map(&:join)
```

### Default UTF-8 encoding ###

You can now use useful characters outside of US-ASCII without magic encoding
comments or inscrutable escape sequences

```ruby
currency = "€"   #=> "€"
```

### Binary string shortcut ###

`String#b` as a shortcut to get an ASCII-8BIT (aka binary) copy of a string

```ruby
s = "foo"
s.encoding     #=> #<Encoding:UTF-8>
s.b.encoding   #=> #<Encoding:ASCII-8BIT>
```

### String#lines, #chars, etc return an Array ###

lines, chars, codepoints, bytes now all return arrays rather then enumerators

```ruby
s = "foo\nbar"
s.lines        #=> ["foo\n", "bar"]
s.chars        #=> ["f", "o", "o", "\n", "b", "a", "r"]
s.codepoints   #=> [102, 111, 111, 10, 98, 97, 114]
s.bytes        #=> [102, 111, 111, 10, 98, 97, 114]
```

These still accept a block for backwards compatibility, but you should use
`#each_line` etc for that use case

The similarly named methods on IO, ARGF, StringIO, and Zlib::GzipReader still
return enumerators, but are deprecated, use the each_ versions.

### __dir__ ###

Returns the file path to the executing file, like \__FILE__, but without the
file name, useful for things like

```ruby
YAML.load_file(File.join(__dir__, "config.yml"))
```

### __callee__ returns the method as called ###

__callee__ is back to returning the name of the method as called, not as
defined with an aliased method. This can actually be useful.

```ruby
def do_request(method, path, headers={}, body=nil)
  "#{method.upcase} #{path}"
end

def get(path, headers={})
  do_request(__callee__, path, headers)
end
alias head get

get("/test")    #=> "GET /test"
head("/test")   #=> "HEAD /test"
```

### regexp engine is changed to Onigmo ###

The is a fork of the Oniguruma regexp engine used by 1.9, with a few more
features. More details at https://github.com/k-takata/Onigmo
The new features seem Perl-inspired, this seems to be a good refrence:
http://perldoc.perl.org/perlre.html

    (?(cond)yes|no)

if cond is matched, then match against yes, if cond is false match against no
cond references a match either by group number or name, or is a
look-ahead/behind

example only matches a trailing cap if there is a leading cap

```ruby
regexp = /^([A-Z])?[a-z]+(?(1)[A-Z]|[a-z])$/

regexp =~ "foo"   #=> 0
regexp =~ "foO"   #=> nil
regexp =~ "FoO"   #=> 0
```

### Hash#default_proc= now accepts nil ###

No longer will you have to call `hash.default = nil` to clear
`hash.default_proc` now your code will actually make sense!

```ruby
hash = {}
hash.default_proc = Proc.new {|h,k| h[k] = []}
hash[:foo] << "bar"
hash[:foo]                                       #=> ["bar"]
hash.default_proc = nil
hash[:baz]                                       #=> nil
```

### Array#values_at returns nil for each value that is out-of-range ###

Previously `#values_at` would behave in an unexpected way when given a range, and
you'd only get a single nil for all out-of-range indexes, now you get one for
each

```ruby
[2,4,6,8,10].values_at(3..7)   #=> [8, 10, nil, nil, nil]
```

### Option for File.fnmatch? to expand braces ###

If for some reason you ever find yourself needing to do shell filename glob
matches in Ruby you'll be happy to know you can now use pattens like {foo,bar}

```ruby
# 3rd argument enables the brace expansion
File.fnmatch?("{foo,bar}", "foo", File::FNM_EXTGLOB)   #=> true
File.fnmatch?("{foo,bar}", "foo")                      #=> false
# or together multiple options old-school C style
casefold_extglob = File::FNM_CASEFOLD | File::FNM_EXTGLOB
File.fnmatch?("{foo,bar}", "BAR", casefold_extglob)    #=> true
```

### Shellwords calls #to_s on arguments ###

`Shellwords#shellescape` and `#shelljoin` will now call `#to_s` on the arguments, this
is particularly useful with Pathname.

```ruby
require "pathname"
require "shellwords"

path = Pathname.new("~/Library/Application Support/").expand_path
Shellwords.shellescape(path)
\#=> "/Users/mat/Library/Application\\ Support"

Shellwords.join(Pathname.glob("/Applications/A*"))
\#=> "/Applications/App\\ Store.app /Applications/Automator.app"
```

### system and exec now close non-standard file descriptors by default ###

when using exec all open files/sockets, other than stdin, stdout and stderr
will be closed for the new process. This was previously and option with
`exec(cmd, close_others: true)` but it's not the default.

### Protected methods treated like private for #respond_to? ###

protected methods are now hidden from `#respond_to`? unless true is passed as
a second argument, just like private methods.

```ruby
class Foo
  protected
  def bar
    "baz"
  end
end

f = Foo.new
f.respond_to?(:bar)         #=> false
f.respond_to?(:bar, true)   #=> true
```

### #inspect no longer calls #to_s ###

Under Ruby 1.9 `#inspect` gained the odd behaviour of delegating to `#to_s` if a
custom `#to_s` method had been defined, this has been removed

```ruby
class Foo
  def to_s
    "foo"
  end
end

Foo.new.inspect   #=> "#<Foo:0x007fb4a2887328>"
```

### LoadError#path ###

Load error now has a `#path` method to retrieve the path of the file that
couldn't be loaded. That was already in the message, but now it's more easily
accessible to code

```ruby
begin
  require_relative "foo"
rescue LoadError => e
  e.message   #=> "cannot load such file -- /Users/mat/Dropbox/foo"
  e.path      #=> "/Users/mat/Dropbox/foo"
end
```

### Process.getsid ###

getsid return the processes session ID. This only works on unix/linux systems

```ruby
Process.getsid   #=> 240
```

### Signal.signame ###

A signame method has been added to get the name for a signal number

```ruby
Signal.signame(9)   #=> "KILL"
```

### Error on trapping signals used internally ###

Signal.trap now raises an ArgumentError if you try and trap :SEGV, :BUS,
:ILL, :FPE, or :VTALRM. These are used internally by Ruby, so you wouldn't be
able to trap them anyway.

### True thread local variables ###

As of Ruby 1.9 `Thread#[]`, `#[]=`, `#keys` and `#key`? would get/set fiber
local variables, Thread now gets the methods `#thread_variable_get`,
`#thread_variable_set`, `#thread_variables`, `#thread_variable`? as equivalents
that are fiber local

Fiber local:

```ruby
b = nil

a = Fiber.new do
  Thread.current[:foo] = 1
  b.transfer
  Thread.current[:foo]
end

b = Fiber.new do
  Thread.current[:foo] = 2
  a.transfer
end

p a.resume   #=> 1
```

Thread local:

```ruby
b = nil

a = Fiber.new do
  Thread.current.thread_variable_set(:foo, 1)
  b.transfer
  Thread.current.thread_variable_get(:foo)
end

b = Fiber.new do
  Thread.current.thread_variable_set(:foo, 2)
  a.transfer
end

p a.resume   #=> 2
```

### Better error on joining current/main thread ###

If you attempt to call `#join` or `#value` on the current or main thread you now
get a ThreadError raised, which inherits from StandardError, rather than
'fatal' which inherits from Exception.

```ruby
begin
  Thread.current.join
rescue => e
  e   #=> #<ThreadError: Target thread must not be current thread>
end
```

### Mutex changes ###

I can't think of a particularly interesting example for this, but you can now
check if the current thread owns a mutex.

```ruby
require "thread"

lock = Mutex.new
lock.lock
lock.owned?                      #=> true
Thread.new {lock.owned?}.value   #=> false
```

Also affecting Mutex, methods that change the state of the mutex are no longer
allowed in signal handlers; `#lock`, `#unlock`, `#try_lock`, `#synchronize`, and
`#sleep`

And apparently `#sleep` may wake up early, so you'll need to double check the
correct amount of time has passed if precise timings are important.

```ruby
sleep_time = 0.1
start = Time.now
lock.sleep(sleep_time)
elapsed = Time.now - start
lock.sleep(sleep_time - elapsed) if elapsed < sleep_time
```

### Custom thread and fiber stack sizes ###

The following environment variables can be set to alter the stack sizes used by
threads and fibers. Ruby only checks these as your program starts up.

RUBY_THREAD_VM_STACK_SIZE: vm stack size used at thread creation.
default: 128KB (32bit CPU) or 256KB (64bit CPU).

RUBY_THREAD_MACHINE_STACK_SIZE: machine stack size used at thread creation.
default: 512KB or 1024KB.

RUBY_FIBER_VM_STACK_SIZE: vm stack size used at fiber creation.
default: 64KB or 128KB.

RUBY_FIBER_MACHINE_STACK_SIZE: machine stack size used at fiber creation.
default: 256KB or 256KB.

you can get the defaults with

```ruby
RubyVM::DEFAULT_PARAMS   #=> {:thread_vm_stack_size=>1048576,
                              :thread_machine_stack_size=>1048576,
                              :fiber_vm_stack_size=>131072,
                              :fiber_machine_stack_size=>524288}
```

### Stricter Fiber#transfer ###

A fiber that has been transferred to now must be transferred back to, instead
of cheating and using resume.

```ruby
require "fiber"

f2 = nil

f1 = Fiber.new do
  puts "a"
  f2.transfer
  puts "c"
end

f2 = Fiber.new do
  puts "b"
  f1.transfer # under 1.9 this could have been a #resume
end

f1.resume
```

### RubyVM::InstructionSequence ###

RubyVM::InstructionSequence isn't new, but it gains a couple of features, and
even more helpfully, [detailed documentation][is_docs]

You can now get the instruction sequence an existing method

```ruby
class Foo
  def add(x, y)
    x + y
  end
end

instructions = RubyVM::InstructionSequence.of(Foo.instance_method(:add))
```

and when you have that instruction sequence you can get some details about
where it was defined

```ruby
instructions.path            #=> "/Users/mat/Dropbox/ruby-2.0.0.rb"
instructions.absolute_path   #=> "/Users/mat/Dropbox/ruby-2.0.0.rb"
instructions.label           #=> "add"
instructions.base_label      #=> "add"
instructions.first_lineno    #=> 654
```

[is_docs]: http://www.ruby-doc.org/core-2.0/RubyVM/InstructionSequence.html

### ObjectSpace::WeakMap ###

This class is mainly intended to be part of WeakRef's implementation, so you
should probably use that (require "weakref"). It holds a weak reference to the
objects stored, which means they may be garbage collected.

```ruby
map = ObjectSpace::WeakMap.new
# keys can't be immediate values (numbers, symbols), and you must use the
# exact same object, not just one that is equal.
key = Object.new

map[key] = "foo"
map[key]                #=> "foo"
# force a garbage collection run
sleep(0.1) and GC.start 
map[key]                #=> nil
```

### top level define_method ###

define_method can now be used at the top level; it doesn't have to be inside a
class or module

```ruby
Dir["config/*.yml"].each do |path|
  %r{config/(?<name>.*)\.yml\z} =~ path
  define_method(:"#{name}_config") {YAML.load_file(path)}
end
```

### No warning for unused variables starting with _ ###

This method will generate warnings that the family, port, and host variables
are unused.

```ruby
def get_ip(sock)
  family, port, host, address = sock.peeraddr
  address
end
```

Using underscores will stop the warnings, but lose the self-documenting nature
of the code

```ruby
def get_ip(sock)
  _, _, _, address = sock.peeraddr
  address
end
```

As of Ruby 2.0.0 we can get the best of both worlds by starting the variables
with an _

```ruby
def get_ip(sock)
  _family, _port, _host, address = sock.peeraddr
  address
end
```

### Proc#== and #eql? removed ###

Under Ruby 1.9.3 Procs with the same body and binding were equal, but you'd
only get procs like this when you'd cloned one from another. This has now been
removed, which is no great loss as it wasn't really very useful.

```ruby
proc = Proc.new {puts "foo"}

proc == proc.clone   #=> false
```

### ARGF#each_codepoint ###

ARGF (which is a concatination of the files supplied on the command line) gets
an `#each_codepoint` method like IO.

```ruby
count = 0
ARGF.each_codepoint {|c| count += 1 if c > 127}
puts "there are #{count} non-ascii chacters in the given files"
```

### Time#to_s ###

The encoding of the string returned from `Time#to_s` changes from ASCII-8BIT
(aka binary) to US-ASCII.

```ruby
Time.now.to_s.encoding   #=> #<Encoding:US-ASCII>
```

### Random parameter of Array#shuffle! and #sample now called with a 'max' arg

This is a small change, that will probably have no affect on you at all, but
now when supplying a random param to the `#shuffle!`, `#shuffle`, and `#sample`
methods on Array it's now expected to take a 'max' argument, and return an
integer between 0 and max, rather than a float between 0 and 1.

```ruby
array = [1, 3, 5, 7, 9]
randgen = Object.new
def randgen.rand(max)
  max                           #=> 4
  1
end
array.sample(random: randgen)   #=> 3
```

### CGI HTML5 tag builder ###

CGI from the stdlib gets an HTML5 mode for its tag builder interface.

```ruby
require "cgi"
cgi = CGI.new("html5")
html = cgi.html do
  cgi.head do
    cgi.title {"test"}
  end +
  cgi.body do
    cgi.header {cgi.h1 {"example"}} +
    cgi.p {"lorem ipsum"}
  end
end
puts html
```

The old `#header` method (to send the HTTP header) is now called `#http_header`,
although as long as you're not in HTML5 mode it's aliased as `#header` for
backwards compatibility.

### CSV::dump and ::load removed ###

`CSV::dump` and `CSV::load` have been removed. They allowed you to dump/load an
array of Ruby objects to a CSV file, and have them serialised and deserialised.
They've been removed as they were unsafe.

### Iconv removed ###

Iconv has been removed, in preference of `String#encode`.

Where previously you might have written something like:

```ruby
require "iconv"
Iconv.conv("ISO-8859-1", "UTF8", "Résumé")   #=> "R\xE9sum\xE9"
```

You'd now write

```ruby
"Résumé".encode(Encoding::ISO_8859_1)        #=> "R\xE9sum\xE9"
```

### Syck removed ###

The Syck YAML parser has been removed in favour of Psych (libyaml bindings),
and Ruby now comes bundled with libyaml. The YAML interface in Ruby stays the
same, so this shouldn't have and affect on your code.

### io/console ###

io/console isn't new, but the [documentation][io_docs] is so now you can
actually figure out how to use it. The Ruby 2.0.0 NEWS file claims the
`IO#cooked` and `#cooked` methods are new, but they seem to be available in
1.9.3.

```ruby
require "io/console"
IO.console.raw!
# console in now in raw mode, disabling line editing and echoing
IO.console.cooked!
# back in cooked mode, line editing works like normal
```

`#raw!` and `#raw` get two new arguments, min and time.

```ruby
IO.console.raw!(min: 5) # reading from console buffers for 5 chars
IO.console.raw!(min: 5, time: 1) # read after 1 second if buffer not full
```

[io_docs]: http://www.ruby-doc.org/stdlib-2.0/libdoc/io/console/rdoc/IO.html

### io/wait ###

io/wait adds a `#wait_writeable` method that will block till an IO can be
written to. `#wait` gets renamed to `#wait_readable`, and there's a `#wait`
alias for backwards compatibility.

```ruby
require "io/wait"
timeout = 1
STDOUT.wait_writable(timeout)   #=> #<IO:<STDOUT>>
```

### Net::HTTP performance improvements ###

Net::HTTP now automatically requests and decompresses gzip and deflate
compression by default. This should play very nicely with the new
non-GIL-blocking Zlib.

SSL sessions are also now reused, cutting down on time spent negotiating
connections.

### Net::HTTP can specify the host/port to connect from ###

If for some reason you need to specify the local host/port to connect from,
along with the host/port to connect to, you now can

```ruby
http = Net::HTTP.new(remote_host, remote_port)
http.local_host = local_host
http.local_port = local_port
http.start do
  # ...
end
```

### OpenStruct can act like a hash ###

OpenStruct gains `#[]`, `#[]=` and `#each_pair` methods so it can be used like
a hash.

```ruby
require "ostruct"

o = OpenStruct.new
o.foo = "test"
o[:foo]               #=> "test"
o[:bar] = "example"
o.bar                 #=> example
```

It also gains `#hash` and `#eql`? methods, which are used internally by Hash to
check equality. These allow it to play better as a hash key, with equal objects
acting as the same key.

### Timeout support in Resolv ###

Resolv now supports custom timeouts

```ruby
require "resolv"

resolver = Resolv::DNS.new
resolver.timeouts = 1 # 1 second
resolver.getaddress("globaldev.co.uk").to_s   #=> "204.232.175.78"
```

It will also take an array, and work its way through the timeouts, retrying
after each. You could implement exponential back-off with

```ruby
resolver.timeouts = [1].tap {|a| 5.times {a.push(a.last * 2)}}
```