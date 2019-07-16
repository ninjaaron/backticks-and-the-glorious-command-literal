First some demonstrations of how commands and/or backticks work in
other languages. I use Bash and Ruby for demonstration, but whatever
is true of Ruby here is basically also true of Perl. I just use Ruby
because it ships with an iteractive prompt.

Hello

```bash
$ echo "hello, bash"
hello, bash
```

```ruby
irb(main):001:0> puts `echo "hello, ruby"`
hello, ruby
```

```julia
julia> run(`echo "hello, julia"`)
hello, julia
```

Whitespace

```bash
$ str=$'hello\nbash'
$ echo $str
hello bash
```

```ruby
irb(main):002:0> str = "hello\nruby"
=> "hello\nruby"
irb(main):003:0> puts `echo #{str}`


# hangs indefinitely
```

```julia
julia> str = "hello\njulia"
"hello\njulia"

julia> run(`echo $str`)
hello
julia
```

Can be fixed like this:
```bash
$ echo "$str"
hello
bash
```

One more:

```bash
$ str='a "goofy string'
$ echo "$str"
a "goofy string
$ # bash gets one right!
```

```ruby
irb(main):006:0> str = 'a "goofy string'
=> "a \"goofy string"
irb(main):007:0> puts `echo "#{str}"`
sh: 1: Syntax error: Unterminated quoted string

=> nil
irb(main):008:0> # ruby not so much
```

```julia
julia> str = "a \"goofy string"
"a \"goofy string"

julia> run(`echo $str`)
a "goofy string
```



(How to do it in ruby...)
```ruby
def shellescape(arg)
  return "'" + arg.gsub("'", "'\\''")  + "'"
end
```

Alternatively, use `popen`: (actually, you should just always use
`popen` to get the output of a 

```ruby
irb(main):029:0> puts IO.popen(["echo", str]).read
a "goofy string
```

Python doesn't have any of these problems. If you want to get a string
from a shell command, simply use this memorable incantation:

```python
>>> import subprocess
>>> string = 'a "goofy string'
>>> import subprocess
>>> print(subprocess.run(["echo", string], universal_newlines=True, stdout=subprocess.PIPE).stdout)
a "goofy string
```

So, what's going on here? In Bash, you must quote every variable. If
you don't it will be expanded into arguments on whitespace. (Note that
ZSH and Fish do _not_ have this insane behavior)

In Ruby (and Perl), a string is built inside the backticks and they
just send that string to a shell. This is the most disasterous
possible behavior. It's an injection vulnerability. Even if you don't
have user input in the string, something like a file name with
characters significant to the shell could break your pipeline unless
you escape it properly.

But what happens in Julia? Though backticks in Julia look
superficially to those in Bash, Perl, Ruby and others, they are
something entirely different.

```julia
julia> str = "foo bar"
"foo bar"

julia> cmd = `echo $str`
`echo 'foo bar'`
```

Backticks in Julia are actually a constructor for a literal object:

```julia
julia> typeof(cmd)
Cmd

julia> fieldnames(ans)
(:exec, :ignorestatus, :flags, :env, :dir)
```

The most important field here is `exec`, which is an array of strings:

```julia
julia> cmd.exec
2-element Array{String,1}:
 "echo" 
 "foo bar"
```
This might perhaps remind us of the first argument taken by
`subprocess.run` in Pyhton or `IO.popen` in Ruby--or perhaps of the
`exec` familiy of funtions in C. Indeed, all of these arrays of
strings eventually are handed off to one of the `exec` functions (on
unix-like systems, anyway), meaning the process is run directly by the
operating system without a shell being invoked. 

The difference with Julia is that it allows you to write the command
more or less the way you would enter it in the shell. The secret here
is that Julia actually contains a parser for a shell-like
mini-language in a file called shell.jl. Yep, Julia's command literals
are implemented in Julia directly. The backtick syntax is simply
transformed into an invocation of the `cmd` macro, which hands the
string off to a function called `shell_parse`.

However, Julia fixes one of the biggest problems with the shell:
expansion of unquoted variables into arguments on whitespace. However,
there are some things which are expanded into arguments: iterables.

```julia
julia> a = 1:3
1:3

julia> `echo $a`
`echo 1 2 3`
```

Like brace expansion in the shell, Julia's shell parser also permits building up
Cartesian products in this way.

```julia
julia> b = 4:6
4:6

julia> # compare with `echo {1..3}{4..6}` in Bash

julia> `echo $a$b`
`echo 14 15 16 24 25 26 34 35 36`
```

Basically, you get the convenience and familiarity of a shell-like
syntax for describing commands, but the safety of avoiding ever
passing any data to the shell.

Julia's shell parser does lack a few features. Specifically, it
doesn't allow use the pipe character and angled brackets to describe
IO redirection. It's pretty easy to do these things with the
`pipeline` function.

```julia
julia> pipeline(`echo foo`, `tr a-z A-Z`) |> run
FOO
Base.ProcessChain(Base.Process[Process(`echo foo`, ProcessExited(0)), Process(`tr a-z A-Z`, ProcessExited(0))], Base.DevNull(), Base.DevNull(), Base.DevNull())
```

However, a little birdie told me that the addition of redirection
operators to Julia's shell parser is under consideration for future
versions of Julia.

There are also some rather pleasant, but possibly unexpected
differences between Julia's process interface and those in other
languages that I find quite pleasant. For one, using "process
substitution syntax", `$()` in Julia does not create a subprocess and
interpolate the output into your command. Instead, it allows you to
interpolate Julia expressions into your commands. This may be a
surprise for shell veterans, but it is typically much more useful (and
way faster) than invoking more processes to do work.

Additionally, by default, processes that exit with anything other than
`0` will _throw and exception_. It's kind of nuts that other process
interfaces don't have this default. Granted, there are times when a
non-zero exit status doesn't mean an error. For example, `grep` exits
with `1` if it doesn't find any matches--not exactly an error.

However, given that non-zero exit status is used to indicate an error
in the vast majority of cases, throwing an exception in a language
like Julia that makes you deal with runtime errors seems like the
right choice. For some reason, Python and Ruby don't.

```julia
julia> run(`cat foo`)
cat: foo: No such file or directory
ERROR: failed process: Process(`cat foo`, ProcessExited(1)) [1]
Stacktrace:
 [1] error(::String, ::Base.Process, ::String, ::Int64, ::String) at ./error.jl:42
 [2] pipeline_error at ./process.jl:785 [inlined]
 [3] #run#515(::Bool, ::Function, ::Cmd) at ./process.jl:726
 [4] run(::Cmd) at ./process.jl:724
 [5] top-level scope at none:0
```

Of course, it is possible change this behavior.

Another convenient feature of Julia's command interface is that every
command which would take a file or filename as an argument for dealing
with IO may also take a command literal for example:

```julia
julia> for line in eachline(`ls`)
           uppercase(line) |> println
       end
BAR
BASH
BAZ
FOO
JULIA
RUBY
TALK.MD
```

In summary, Julia has:

- Literal syntax for creating abstract command objects.
- A built-in parser for shell-like syntax.
- Assurance that your commands are safe from injection.*
- A host of helpful interfaces to make running commands and dealing
  with their IO simple.

\* OK, you can have injection if you are using a process that creates a
shell. i.e. invoking Bash directly or using another program like `ssh`
or `rsync` that creates a shell on the remote host. use
`Base.shell_escape` to ensure everything is properly quoted on the
other side.

I believe this combination of features makes Julia a nearly ideal
language for handling processes. Indeed, translating a Bash script
into Julia can often be done trivially, and the result will typically
be safer than the original script.

A final, almost side note about the process interface in Julia is
this: Processes are always run in the event loop. This means they will play
nicely with asynchronous code. Even if you're waiting on the process
to exit in the immediate context of the calling function, you can
later incorporate this function into tasks without worrying about
blocking. Julia is just as asynchronous as JavaScript--and indeed, is
built with the same libuv event loop--but provides non-blocking behavior
without making you care about which functions are asynchronous.

If you're interested in using Julia for administrative scripting as a
safe and easy alternative to Bash (a task for which is surprisingly
well-suited), you might want to look at my tutorial on github, which
provides more information about this topic, as well as information
about working with files and the filesystem, making command-line
utilities with Julia, and some notes about Julia's regular expression
interface.

https://github.com/ninjaaron/administrative-scripting-with-julia
