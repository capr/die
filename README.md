## What

A basic set of functions for flow control and user interaction for shell scripts.
It improves the readability of scripts by making them shorter and more illustrative of author's intent, while at the same time providing a framework for allowing the user of the script to control the interaction. All in 6 lines of code.

```
debug "script started"            # use --debug to see the message
say "Hi"                          # suppress with --quiet
some_cmd || die "can't go on"     # laments and exits with code 1 if the command fails 
run somecmd args                  # use --debug to see the full command and its exit code 
must somecmd args                 # if somecmd fails, exits showing what ran and the exit code
hold "Look at me now"             # asks for a keypress; suppress with --yes or --quiet
error "nothing serious"           # reports an error and continues; supress with --quiet
say "Bye"                         # suppress with --quiet
debug "script ended"              # see this with --debug
```

This is how a die-enahnced script would look like:

```bash
. die

while [ $# -gt 0 ]; do
  case "$1" in
	 --debug) DEBUG=true ;;
	 --quiet) QUIET=true ;;
	 --yes)   YES=true ;;
  esac
  shift
done

say "Good news everyone. Pass --quiet so I won't bother you like this."

foo && bar && baz || die "Oh my."

hold "I need you to press a key. Pass --yes so I won't hold you like this."

run optional_command || error "this didn't work but we go on."

must important_command # if this fails, abort the mission.

debug "this is what I just did... blah blah blah... and then..."

```

## Why

Adding error checking and progress/error reporting to shell scripts makes them ugly to the point where is hard to see the bits that do essential work from the error checking/flow control ones. Encapsulating these cross-cutting concerns aside into a small vocabulary solves the problem at the expense of learning the vocabulary.

## The Code

```bash
#!/bin/sh (source it!)
# die: basic vocabulary for flow control and progress/error reporting
# v1.0 | Cosmin Apreutesei (public domain) | http://github.com/capr/die
# these functions are influenced by $QUIET, $DEBUG and $YES variables

say()   { [ "$QUIET" ] || echo "$@" >&2; }
error() { say -n "ERROR: "; say "$@"; return 1; }
die()   { echo -n "EXIT: " >&2; echo "$@" >&2; exit 1; }
debug() { [ -z "$DEBUG" ] || echo "$@" >&2; }
run()   { debug -n "EXEC: $@ "; "$@"; local ret=$?; debug "[$ret]"; return $ret; }
must()  { debug -n "MUST: $@ "; "$@"; local ret=$?; debug "[$ret]"; [ $ret == 0 ] || die "$@ [$ret]"; }
hold()  { [ $# -gt 0 ] && say "$@"; [ "$YES$QUIET" ] && return; echo -n "Press ENTER to continue, or ^C to quit."; read; }
```
[source](https://raw.github.com/capr/die/master/die) | [testing unit](https://raw.github.com/capr/die/master/die-test)

## In English

**say** `...`         --- echoes arguments to stderr, and only if `$QUIET` is not set <br>
**error** `...`       --- says `ERROR: ...` <br>
**die** `...`         --- says `EXIT: ...` and exits the current process <br>
**debug** `...`       --- says `...`, but only if `$DEBUG` is set <br>
**run** _cmd_ `...`   --- run a command and debug-say `EXEC: cmd ... [exit code]` <br>
**must** _cmd_ `...`  --- run a command and debug-say `MUST: cmd ... [exit code]`, but die if the command's exit code != 0 <br>
**hold** `...`        --- say `...` and then ask for a key press, but only if `$YES` or `$QUIET` is not set

## Caveats

`die` and `must` can only kill the process they were executed in. This means they won't kill the script if invoked from a subprocess. This can look counterintuitive sometimes:

```bash
somevar="$(must false)"
find | must false
find | while read f; do die "somth bad happen"; done
find | { die "somth bad happen"; } #the pipe created the subprocess, not the braces
(must false)
(false || die "somth bad happen")
echo "none of these killed me"
```
Note that `$somevar` was set to `""`, since `must` complained on stderr. 

Anyway, the way to fix this is to check on the exit code of the subprocess:

```bash
somevar="$(must false)" || die "subprocess failed, echo told us"
find | must false || die "subprocess failed, | tells us"
find | while read f; do die "somth bad happen"; done || die "subprocess failed, I fail"
find | { die "somth bad happen"; } || die "subprocess failed, I fail"
(must false) || die "subprocess failed"
(false || die "somth bad happen") || die "subprocess failed"
echo "can't reach here"
~~~

`hold` won't work in a subprocess that redirects the standard input -- not only that, it will eat one line of the input as well!

```bash
find | hold "I can't hold you, see?"
{ :; } | hold "Even when there's no input I still can't hold you"
~~~

## Feedback

  * `cosmin.apreutesei@gmail.com`

