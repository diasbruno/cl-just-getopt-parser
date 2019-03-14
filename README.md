Just Getopt Parser
==================

**Getopt-like command-line parser for the Common Lisp language**


Introduction
------------

This Common Lisp package implements Unix Getopt command-line parser.
Package's main interface is `getopt` function which parses the command
line options and organizes them to valid options, other arguments and
unknown arguments. For full documentation on package's programming
interface see section _Interface (API)_ down below.


Examples
--------

Example command line:

    $ some-program -d3 -f one --file=two -xyz foo --none bar -v -- -v

That command line could be parserd with the followin function call:

    (getopt '("-d3" "-f" "one" "--file=two" "-xyz" "foo"
              "--none" "bar" "-v" "--" "-v")
            '((:debug #\d :optional)
              (:file #\f :required)
              (:file "file" :required)
              (:verbose #\v))
            :options-everywhere t)

The function returns three values: (1) valid options and their
arguments, (2) other arguments and (3) unknown options:

    ((:DEBUG . "3") (:FILE . "one") (:FILE . "two") (:VERBOSE))
    ("foo" "bar" "-v")
    (#\x #\y #\z "none")

In programs it is probably the most convenient to call this function
through `cl:multiple-value-bind` so that the return valuas are bound to
different variables:

    (multiple-value-bind (options other unknown)
        (getopt COMMAND-LINE-ARGUMENTS '(...))

      ...
      )

In practice there is probably also `cl:handler-bind` macro which handles
error conditions by printing error messages, invoking `continue`
restarts or transferring the program control elsewhere. Here is more
thorough example:

    (handler-bind
        ((ambiguous-option
           (lambda (condition)
             (format *error-output* "~A~%" condition)
             (invoke-restart 'continue)))
         (unknown-option
           (lambda (condition)
             (format *error-output* "~A~%" condition)
             (invoke-restart 'continue)))
         (required-argument-missing
           (lambda (condition)
             (format *error-output* "~A~%" condition)
             (exit-program :code 1)))
         (argument-not-allowed
           (lambda (condition)
             (format *error-output* "~A Skipping the option.~%" condition)
             (invoke-restart 'continue))))

      (multiple-value-bind (options other unknown)
          (getopt COMMAND-LINE-ARGUMENTS '(...)
                  :prefix-match-long-options t
                  :error-on-ambiguous-option t
                  :error-on-unknown-option t
                  :error-on-argument-missing t
                  :error-on-argument-not-allowed t)

        ...
        ))


Author and License
------------------

Author:  Teemu Likonen <<tlikonen@iki.fi>>

PGP: [4E10 55DC 84E9 DFF6 13D7 8557 719D 69D3 2453 9450][PGP]

No restrictions for use: this program is placed in the public domain.

This program is distributed in the hope that it will be useful, but
WITHOUT ANY WARRANTY; without even the implied warranty of
MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.

[PGP]: http://www.iki.fi/tlikonen/pgp-key.asc


Interface (API)
---------------

### Condition: `ambiguous-option`

GETOPT function may signal this condition when it parses a
partially-written that matches two or more long option names. Function
OPTION-NAME can be used to read option's name.


### Condition: `argument-not-allowed`

GETOPT function may signal this condition when it parses an option
that does not allow an argument but one is given with "--foo=...".
Function OPTION-NAME can be used to read option's name.


### Function: `getopt`

The lambda list:

     (arguments option-specification &key options-everywhere
      prefix-match-long-options error-on-ambiguous-option
      error-on-unknown-option error-on-argument-missing
      error-on-argument-not-allowed)

Parse command-line arguments like getopt.

#### Function's arguments

The ARGUMENTS is a list of strings and contains the command-line
arguments that typically come from program's user.

OPTION-SPECIFICATION argument is the specification of valid command-line
options. It is a list that contains lists of the following format (in
lambda list format):

    (SYMBOL OPTION-NAME &optional OPTION-ARGUMENT)

The first element SYMBOL is any symbol which identifies this
command-line option (for example keyword symbol :HELP). The identifier
is used in function's return value to identify that this particular
option was present in the command line.

The second element OPTION-NAME is either

 1. a character specifying a short option name (for example #\h,
    entered as "-h" from command line)

 2. a string specifying a long option (for example "help", entered as
    "--help" from command line). The string must be at least two
    character long.

The third element is optional but if it is non-NIL it must be one of the
following keyword symbols. :REQUIRED means that this option requires an
argument. :OPTIONAL means that this option has an optional argument.
Example value for this function's OPTIONS argument:

    ((:HELP #\h)     ; short option -h for help (no option argument)
     (:HELP "help")  ; long option --help (no option argument)
     (:FILE "file" :REQUIRED) ; --file option which requires argument
     (:DEBUG #\d :OPTIONAL))  ; -d option with optional argument

Note that several options may have the same identifier SYMBOL. This
makes sense when short and long option represent the same meaning. See
the :HELP keyword symbol above. All options must have unique OPTION-NAME
though.

If function's key argument OPTIONS-EVERYWHERE is NIL (the default) the
option parsing stops when the first non-option argument is found. Rest
of the command line is parsed as non-options. If OPTIONS-EVERYWHERE is
non-NIL then options can be found anywhere in the command line, even
after non-option arguments. In all cases the option parsing stops when
the pseudo-option "--" is found in the command line. Then all
remaining arguments are parsed as non-option arguments.

If key argument PREFIX-MATCH-LONG-OPTIONS is non-NIL then long options
don't need to be written in full in the command line. They can be
shortened as long as there are enough characters to find unique prefix
match. If there are more than one match the option is classified as
unknown. If also key argument ERROR-ON-AMBIGUOUS-OPTION is non-NIL the
function will signal error condition AMBIGUOUS-OPTION. The condition
object contains the option's name and it can be read with function
call (OPTION-NAME <condition>). Function call (OPTION-MATCHES
<condition>) returns a list of option matches (strings). Also, the
condition object can be printed as an error message for user. There is
also CONTINUE restart available. When invoked the ambiguous option is
skipped and the function will continue parsing the command line.
Ambiguous options are always also unknown options: if AMBIGUOUS-OPTION
condition is not signaled then the condition for unknown option can be
signaled. See the next paragraph.

If function's key argument ERROR-ON-UNKNOWN-OPTION is non-NIL and the
function finds an uknown option on the command line the function signals
error condition UNKNOWN-OPTION. The condition object includes the name
of the unknown option which can be read with function (OPTION-NAME
<condition>). The return value is of type character or string for short
or long options respectively. You can also just print the condition
object: it gives a reasonable error message. There is also CONTINUE
restart available. The invoked restart skips the unknown option and
continues parsing the command line.

Function's key argument ERROR-ON-ARGUMENT-MISSING, if non-NIL, causes
the function to signal error condition REQUIRED-ARGUMENT-MISSING if it
sees an option which required argument (keyword :REQUIRED) but there is
none. The condition object contains the name of the option which can be
read with function (OPTION-NAME <condition>). You can also just print
the condition object for user. It's the error message. There are two
restarts available: CONTINUE restart accepts the missing argument (value
NIL) and continues with the parser; GIVE-ARGUMENT restart must be
invoked with a new argument (string) for the option.

Key argument ERROR-ON-ARGUMENT-NOT-ALLOWED, if non-NIL, makes this
function to signal error condition ARGUMENT-NOT-ALLOWED if there is an
argument for a long option which does not allow argument (--foo=...).
Such option is always listed as unknown option with name "foo=" in
function's return value. The condition object can be printed to user as
error message. The object also contains the name of the option which can
be read with (OPTION-NAME <condition>) function call. There is CONTINUE
restart available. When invoked the function continues parsing the
command line.


#### Return values

The function returns three values:

 1. List of parsed options. List's items are cons cells: the CAR part of
    the cons cell is the identifier symbol for the option; the CDR part
    of the cons cell is either NIL (if there is no argument for this
    option) or a string containing option's argument.

 2. List of non-option arguments (strings).

 3. List of unknown options. List's items are either characters or
    strings which represent command-line options which were not defined
    in the OPTIONS specification.

In all return values the list's items are in the same order as they were
in the original command line.


#### Parsing rules for short options

Short options in the command line start with the "-" character and the
option character follows (-c).

If option requires an argument (keyword :REQUIRED) the argument must be
entered either directly after the option character (-cARG) or as the
next command-line argument (-c ARG). In the latter case anything that
follows -c will be parsed as option's argument.

If option has optional argument (keyword :OPTIONAL) it must always be
entered directly after the option character (-cARG). Otherwise there is
no argument for this option.

Several short options can be entered together after one "-"
character (-abc) but then only the last option in the series may have
required or optional argument.


#### Parsing rules for long options

Long options start with "--" characters and the option name comes
directly after it (--foo).

If option requires an argument (keyword :REQUIRED) it must be entered
either directly after the option name and "=" character (--foo=ARG) or
as the next command-line argument (--foo ARG). In the latter case
anything that follows --foo will be parsed as its argument.

If option has optional argument (keyword :OPTIONAL) the argument must
always be entered directly after the option name and "="
character (--foo=ARG). Otherwise (like in --foo) there is no argument
for this option.

Option --foo= is valid format when option has required or optional
argument. It means that the argument is empty string.


### Condition: `required-argument-missing`

GETOPT function may signal this condition when it parses an option
that required an argument but there is none. Function OPTION-NAME can be
used to read option's name.


### Condition: `unknown-option`

GETOPT function may signal this condition when it find an unknown
condition. Function OPTION-NAME can be used to read option's name.


