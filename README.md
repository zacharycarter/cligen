cligen: A Native API-Inferred Command-Line Interface Generator For Nim
======================================================================
This approach to CLIs comes from Andrey Mikhaylenko's nice Python argh module.
Much as with Python, an intuitive subset of ordinary Nim calls maps cleanly
onto command calls, syntactically and semantically.  That subset is basically
any proc with A) all parameters having default values with B) one sequence
parameter identified to capture the positional argument list and C) a return
type of int or no return type at all.  A proc definition following this
"command-like convention" directly implies a command-line interface which
`cligen` can generate automatically.

This approach has both great Don't Repeat Yourself ("DRY", or relatedly "a few
points of edit") properties.  It also has nice "loose coupling" properties.
`cligen` need not even be *present on the system* unless you are compiling a
CLI executable.  Conversely, the wrapped routine does not need to be in the
same module or even a writable file or know anything about `cligen`.  The
learning curve/cognitive load is all about as painless as possible - mostly
just learning what sort of proc is "command-like" enough (and various more
minor controls).  This approach really shines when you want to maintain an
API/CLI in parallel.

Enough Background..Get To The Good Stuff!
-----------------------------------------
In Nim terms, adding a CLI can be as easy as:
```nim
proc foobar(foo=1, bar=2.0, baz="hi", verb=false, paths: seq[string]): int =
  ##Some existing API call with all params defaulted but for a final seq[string]
  result = 1          # Of course, real code would have real logic here

when isMainModule:
  import cligen
  dispatch(foobar)    # Yep..It can really be this simple!
```
Compile it to foobar (assuming nim c foobar.nim is appropriate, say) and then
run ./foobar --help to get a minimal (but not so useless) help message:
```
Usage:
  foobar [optional-params] [paths]
Options:
  --help, -?                    print this help message
  --foo=, -f=   int     1       set foo
  --bar=, -b=   float   2.0     set bar
  --baz=        string  "hi"    set baz
  --verb, -v    toggle  false   set verb
```
Other invocations (foobar --foo=2 --bar=2.7 ...) all work as you would expect.

When you feel like producing a better help string, tack on some parameter-keyed
metadata with Nim's association-list literals and maybe throw in an more overall
description of operation doc string for before the options table:
```nim
  dispatch(foobar, doc = "Deletes no positional-params!",
           help = { "foo" : "the beginning", "bar" : "the rate" })
```
If you want to manually control the short option for a parameter, you can
just override it with the 5th|short= macro parameter:
```nim
  dispatch(foobar, short = { "bar" : 'r' }))
```
With that, "bar" will get 'r' while "baz" will get 'b'.

If you don't like the help message as-is, you can re-order it however you like
with some named-argument string interpolation:
```nim
  dispatch(foobar,          # swap place of doc string and options table
           usage="Use:\n$command $optPos\nOptions:\n$options\n$doc\n",
           prefix="   "))   # indent the whole message a few spaces.
```

The same basic command-argument-to-native type converters that are used for
option values will be applied to convert positional arguments to seq[T] values:
```nim
proc foobar(mynums: seq[int], foo=1, bar=2.0, verb=false): int =
  ##Some API call with all params defaulted but for an initial seq[int]
  result = 1          # Of course, real code would have real logic here
when isMainModule:
  import cligen; dispatch(foobar)
```
That's basically it.  If an all-defaulted-but-for-a-seq[T] entry point is not
already part of your API, you will have to add one in ordinary Nim style and
`dispatch()` that.  Many users who have read this far can start using `cligen`
without further delay.  The rest of this document may be useful later, though.

Basic Requirements For A Proc To Have A Well-Inferred Command
=============================================================
There are only a few very easy rules to learn, two of which are obvious enough
to not even warrant Natural numbers ;-), two of which you may have intuited
already by example/off-hand mention, and the last of which you could guess:

 0a. Optional params get default values (Um, how else can the be optional?)
   
 0b. No parameter of a wrapped proc can can be named "help" (collision!)
   
 1. Zero or one params has explicit type seq[T] to catch non-option arguments.

 2. Wrapped procs must have no return or a return type convertible to int.
   
 3. All param types used must have argParse, argHelp support (see Extending..)
    This includes the type T in seq[T] for non-option/positionals.

That's about it.  `cligen` supports the most likely Nim types (int, float, ..)
out of the box, and the system can be extended pretty easily.  Elaboration on
these rules may be helpful when/if you run into harder cases.

Forbidding positional command arguments (more on Rule 1)
-------------------------------------------------------
When there is no explicit `seq[T]` parameter, `cligen` infers that only optional
command parameters are legal.  The name of the seq parameter does not matter,
only that it's type slot is non-empty and a seq[].  When there is no positional
argument catcher, providing non-option arguments is a command syntax error and
reported as such.  This non-option syntax error also commonly occurs when a
command user forgets the [:|=] to separate an option and its value.  Nim's
parsopt2, the current `cligen` backend, requires such separators.  It's easy to
forget since many other option parsers do not require separators, especially
for short options.

Exit Code Behavior (more on Rule 2)
-----------------------------------
Commands/programs/processes return integer codes to indicate exit status (only
the lowest order byte is significant on many OSes).  All command-line syntax
errors cause programs to exit with status 1.  If there is no return type then
zero is returned to indicate a successful exit (unless an exception is thrown).
If the return type of the wrapped proc is not int, Nim will try to apply any
in-scope converter.  If there is no converter toInt(rtype) the macro will fail
with an error of the form "got (rtype) but expected 'int'" with the line number
of macro invocation.

While there may be some "when compiles()" magic that could fall back to discard
the return value and return zero for non-convertible-to-int's, it may be wiser
to make dispatch() users think a little about mapping non-integer return values
to exit codes.  Defining a little wrapper proc that returns an int (or has no
return) may be easier or clearer than defining a converter, but that does mean
when you add a parameter to your entry point proc you have to add it in two
places which isn't very DRY.  Meh.  If popular demand ensues, discarding
non-convertible-to-int's isn't so hard or so bad.  Another future direction
might be to echo the result to the standard output (perhaps just for `string`
return types).

Extending `cligen` to support new optional parameter types (more on Rule 3)
---------------------------------------------------------------------------
You can extend the set of supported types by defining a couple helper templates
before invoking `dispatch`.  All you need do is define a compatible `argParse`
and `argHelp` for any new Nim parameter types you want.  Basically, `argParse`
parses a string into a Nim value and `argHelp` provides simple guidance on what
that syntax is for command users.

For example, you might want to receive a `seq[string]` parameter inside a single
argument/option value.  So, you need some user friendly convention to convert
a single string to a sequence of them, such as a comma-separated-value list.

Teaching `cligen` what to do goes like this:
```nim
proc demo(stuff = @[ "abc", "def" ], opt1=true, foo=2): int =
  return len(stuff)

when isMainModule:
  import strutils, cligen, argcvt  # argcvt.keys deals with missing short opts

  template argParse(dst: seq[string], key: string, val: string, help: string) =
    dst = val.split(",")

  template argHelp(helpT: seq[array[0..3, string]], defVal: seq[string],
                   parNm: string, sh: string, parHelp: string) =
    helpT.add([keys(parNm, sh), "CSV", "\"" & defVal.join(",") & "\"", parHelp])

  dispatch(demo, doc="NOTE: CSV=comma-separated value list")
```
Of course, you often want more input validation than this.  See `argcvt.nim` in
the `cligen` package for the currently supported types and more details.

Note also that, since `stuff` is a `seq` and there can be only one `seq[T]` for
positionals, type inference for `stuff=@[...]` in `demo` is required.  Using
`(stuff: seq[string] = @[...],...)` would yield either an error or unintended
syntax (`command --foo=3 "a,b,c" "d,e,f"` rather than `--stuff="a,b,c"`).

Related Work
============
`docopt` has similar DRY features and provides superior control over help
messages and richer command line syntax -- mutually exclusive choices and such.
Basically, `docopt` is a command-centric CLI framework while `cligen` is more
API-centric.  The downside to `docopt` is learning its "code-ized" POSIX help
string syntax and having "less natural" parameter access/docopt API calls all
over user code.  There are surely pros and cons.  Implied/inferred CLIs also
require a stronger programming language (at least parameter defaults).  `cligen`
encourages people to preserve "Nim import access" to provided functionalities.
Simple uses get simple commands.  Complex usages and composition with other
code can be driven by Nim programs rather than messier shell scripts (once the
complexity makes command script/shell language limitations bothersome).

Future directions/TODO
======================
I felt `cligen` was useful enough right now to release.  That said..the TODO is
long-ish and includes what many might deem "basic features" :-)

 - Might be nice to be able to pass through (from dispatch) colGap, min4th, and
   maybe a new param to double-space optionally (extra \n between optTab rows).
   [dispatch getting to be a pretty fat interface, but formatting usually is.]

 - Better error reporting. E.g., help={"foo" : "stuff"} silently ignores "foo"
   if there is no such parameter.  Etc.

 - Automate git/nimble-like multi-dispatch. Not so bad..See test/ManualMulti.nim
   Because subcmds are a hard boundary in argv, this needs to do its own opt/arg
   parsing and hard-stop at the first non-option or at least valid subcmd names.

 - Would be nice to provide control over option parsing backend instead of just
   always using parseopt2. Can roll own more traditionally Unix-y backend.  Can
   infer that only bool options can not expect arguments, and can be combined
   like "ls -lt" while non-bool options require vals and need no :|= separator.

 - Make un-defaulted non-seq proc params into mandatory command args in-order
   and converted, binding leftovers to the seq[T]?  Better user error msgs for
   positional argParse failures.  Could also use argv "--" separator to allow
   multiple positional sequences.

 - It might be nice to provide control over what dialect is used to translate
   "multiWord" parameter idents command syntax ("--multi-word", --multi_word,..)
   or maybe most Nim-like to just accept all such dialects? { Maybe all that's
   needed is a strutils.normalize() that takes out "-" as well as "_". }

 - In Nim, `##`-doc comments are the norm rather than doc strings as in Python.
   Pragma macros can get the comment text, but getImpl cannot due to .comment
   not being copied around.  https://github.com/nim-lang/Nim/issues/3690
   Araq favors fixing propagation (but also a bigger .comment->.strVal change).
   Once resolved, we can default doc=ThatText.  In Nim, ThatText usually will
   describe overall operation and parameter semantics.  This implies that in a
   very common-case merely dispatch(myapi) would be all that was needed.
