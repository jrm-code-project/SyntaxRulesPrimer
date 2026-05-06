  # JRM's Syntax-rules Primer for the Merely Eccentric

  In learning to write Scheme macros, I have noticed that it is easy to find
both trivial examples and extraordinarily complex examples, but there seem to be
no intermediate ones. I have discovered a few tricks in writing macros and
perhaps some people will find them helpful.

  The basic purpose of a macro is **syntactic** abstraction. As functions allow
you to extend the functionality of the underlying Scheme language, macros allow
you to extend the syntax. A well-designed macro can greatly increase the
readability of a program, but a poorly designed one can make a program
completely unreadable.

  Macros are also often used as a substitute for functions to improve
performance. In an ideal world, this would be unnecessary, but compilers have
limitations and a macro can often provide a workaround. In these cases, the
macro should be a "drop-in" replacement for the equivalent function, and the
design goal is not to extend the syntax of the language but to mimic the
existing syntax as much as possible.

  ## Very Simple Macros

  `SYNTAX-RULES` provides very powerful pattern-matching and destructuring
facilities. With very simple macros, however, most of this power is unused. Here
is an example:

```scheme
(define-syntax nth-value
  (syntax-rules ()
    ((nth-value n values-producing-form)
     (call-with-values
       (lambda () values-producing-form)
       (lambda all-values
         (list-ref all-values n))))))
```

  When using functions that return multiple values, it is occasionally the case
that you are interested in only one of the return values. The `nth-value` macro
evaluates the `values-producing-form` and extracts the Nth return value.

  Before the macro has been evaluated, Scheme would treat a form that begins
with `nth-value` as it would any other form: it would look up the value of the
variable `nth-value` in the current environment and apply it to the values
produced by evaluating the arguments.

  `define-syntax` introduces a new special form to Scheme. Forms that begin with
`nth-value` are no longer simple procedure applications. When Scheme processes
such a form, it uses the `syntax-rules` we provide to rewrite the form. The
resulting rewrite is then processed in place of the original form.

  > *** Forms are only rewritten if the operator position is an IDENTIFIER that
has been `define-syntax`'d. Other uses of the identifier are not rewritten.

  You cannot write an "infix" macro, nor can you write a macro in a "nested
position," i.e., an expression like `((foo x) y z)` is always considered a
procedure application. (The subexpression `(foo x)` will be rewritten of course,
but it will not be able to affect the processing of subexpressions `y` and `z`.)
This will be important later on.

  `syntax-rules` is based on token-replacement. `syntax-rules` defines a series
of patterns and templates. The form is matched against the pattern and the
various pieces are transcribed into the template. This seems simple enough, but
there is one important thing to always keep in mind:

  > *** THE SYNTAX-RULES SUBLANGUAGE IS NOT SCHEME!

  This is a crucial point that is easy to forget. In the example:

```scheme
(define-syntax nth-value
  (syntax-rules ()
    ((nth-value n values-producing-form)
     (call-with-values
       (lambda () values-producing-form)
       (lambda all-values
         (list-ref all-values n))))))
```

  the pattern `(nth-value n values-producing-form)` looks like Scheme code and
the template `(call-with-values (lambda () values-producing-form) (lambda
all-values (list-ref all-values n)))` **really** seems to be Scheme code, but
when Scheme is applying the `syntax-rules` rewrite there is **NO SEMANTIC
MEANING** attached to the tokens. The meaning will be attached at a later point
in the process, but not here.

  One reason this is easy to forget is that in a large number of cases it
doesn't make a difference, but when you write more complicated rules you may
find unexpected expansions. Keeping in mind that `syntax-rules` only manipulates
patterns and templates will help avoid confusion.

  This example makes good use of patterns and templates. Consider the form:
`(nth-value 1 (let ((q (get-number))) (quotient/remainder q d)))`

  During the expansion of `nth-value`, the **entire** subform `(let ((q
(get-number))) (quotient/remainder q d))` is bound to `values-producing-form`
and transcribed into the template at `(lambda () values-producing-form)`. The
example would not work if `values-producing-form` could only be bound to a
symbol or number.

  A pattern consists of a symbol, a constant, a list (proper or improper) or
vector of more patterns, or the special token "...". (A series of three
consecutive dots in this paper will **always** mean the literal token "..." and
**never** be used for any other reason.) It is not allowed to use the same
symbol twice in a pattern. Since the pattern is matched against the form, and
since the form **always** starts with the defined keyword, it does not
participate in the match.

  You may reserve some symbols as "literals" by placing them in the list that is
the first argument to `syntax-rules`. Essentially, they will be treated as if
they were constants but there is some trickiness that ensures that users of the
macro can still use those names as variables. The trickiness "does the right
thing" so I won't go into details.

  You may find macros written using the token `_` rather than repeating the name
of the macro:

```scheme
(define-syntax nth-value
  (syntax-rules ()
    ((_ n values-producing-form)
     (call-with-values
       (lambda () values-producing-form)
       (lambda all-values
         (list-ref all-values n))))))
```

  I personally find this to be confusing and would rather duplicate the macro
name.

  ### Rules for pattern matching:

  * A constant pattern will only match against an `equal?` constant. * A symbol
that is one of the "literals" can only match against the exact same symbol in
the form, and then only if the macro user hasn't shadowed it. * A symbol that is
**not** one of the literals can match against **any** complete form. (Forgetting
this can lead to surprising bugs.) * A proper list of patterns can only match
against a list form of the same length and only if the subpatterns match. * An
improper list of patterns can only match against a list form of the same or
greater length and only if the subpatterns match. The "dotted tail" of the
pattern will be matched against the remaining elements of the form. It rarely
makes sense to use anything but a symbol in the dotted tail. * The `...` token
is special.

  ## Debugging Macros

  As macros get more complicated, they become trickier to debug. Most Scheme
systems have a mechanism by which you can invoke the macro expansion system on a
piece of list structure and get back the expanded form.

  In MzScheme:
```scheme
(syntax-object->datum
   (expand '(nth-value 1 (let ((q (get-number)))
                           (quotient/remainder q d)))))
```

  In MIT Scheme:
```scheme
(unsyntax (syntax '(nth-value 1 (let ((q (get-number)))
                                  (quotient/remainder q d)))
                   (nearest-repl/environment)))
```

  Be prepared for some interesting output—you may not realize how many forms are
really macros and how much code is produced. The macro system may recognize and
optimize certain patterns of function usage as well. It would not be unusual to
see `(caddr x)` expand into `(car (cdr (cdr x)))` or into `(%general-car-cdr 6
x)`.

  Be prepared, too, for some inexplicable constructs. Some syntax objects may
refer to bindings that are only lexically visible from within the expander.
Syntax objects may contain information that is lost when they are converted back
into list structure. You may encounter apparently illegal expansions like this:

  `(lambda (temp temp temp) (set! a temp) (set! b temp) (set! c temp))`

  There are three internal syntax objects that represent the three different
parameters to the lambda expression, and each assignment referred to a unique
one, but each individual syntax object had the same symbolic name, so their
unique identity was lost when they were turned back into list structure.

  ### Debugging Trick 1 Wrap the template with a quote:

```scheme
(define-syntax nth-value
  (syntax-rules ()
    ((_ n values-producing-form)
     '(call-with-values
        (lambda () values-producing-form)
        (lambda all-values
          (list-ref all-values n))))))
```

  Now the macro returns the filled-in template as a quoted list: `(nth-value
(compute-n) (compute-values))` => `(call-with-values (lambda ()
(compute-values)) (lambda all-values (list-ref all-values (compute-n))))`

  ### Debugging Trick 2 Write a debugging template:

```scheme
(define-syntax nth-value
   (syntax-rules ()
     ((_ n values-producing-form)
      '("Debugging template for nth-value"
        "n is" n
        "values-producing-form is" values-producing-form))))
```

  Sometimes it is difficult to understand why a pattern didn't match something
you thought it should or why it did match something it shouldn't. It is easy to
write a pattern testing macro:

```scheme
(define-syntax test-pattern
  (syntax-rules ()
    ((test-pattern one two) "match 1")
    ((test-pattern one two three) "match 2")
    ((test-pattern . default) "fail")))
```

  ## N-ary Macros

  By using a dotted tail in the pattern we can write macros that take an
arbitrary number of arguments.

```scheme
(define-syntax when
  (syntax-rules ()
    ((when condition . body) (if condition (begin . body) #f))))
```

  An example usage:
```scheme
(when (negative? x)
   (newline)
   (display "Bad number: negative."))
```

  The pattern matches as follows: `condition = (negative? x)` `body = ((newline)
(display "Bad number: negative."))`

  Since the pattern variable `body` is in the dotted tail position, it is
matched against the list of remaining elements. This can lead to errors if
written improperly:

```scheme
;; Incorrect version
(define-syntax when
  (syntax-rules ()
    ((when condition . body) (if condition (begin body) #f))))
```

  This expands to `(if (negative? x) (begin ((newline) (display "Bad number:
negative."))) #f)`. This **almost** works because the consequence is evaluated
as a function call, but it’s logically flawed.

  > *** Macro "rest args" get bound to a list of forms, so remember to "unlist"
them at some point.

  There is another bug in the original form: `(when (< x 5))` expands into `(if
(< x 5) (begin) #f)`. Recall that pattern variables match anything, including
the empty list. The pattern variable `BODY` is bound to the empty list,
resulting in an illegal `(BEGIN)` form.

  A solution is to modify the macro:
```scheme
(define-syntax when
  (syntax-rules ()
    ((when condition form . forms)
     (if condition (begin form . forms) #f))))
```

  ### Implicit Begin Idiom Use this idiom for a macro that allows an arbitrary
number of subforms and processes them sequentially: 1. The pattern should end in
`FORM . FORMS)` to ensure a minimum of one subform. 2. The template has either
`(begin form . forms)` or uses the implicit begin of another special form (e.g.
`(lambda () form . forms)`).

  ## A Strange and Subtle Pitfall

  Suppose you wish to write a macro `PLEASE` that simply removes itself from the
expansion: `(please display "foo") => (display "foo")`

```scheme
;; Faulty attempt
(define-syntax please
  (syntax-rules ()
    ((please . forms) forms)))
```

  This works on some Scheme systems but fails on others (like MzScheme) because
the environment of the pattern matcher does not have a syntactic mapping to
interpret a returned list as a function call.

  The fix is trivial:
```scheme
(define-syntax please
  (syntax-rules ()
    ((please function . arguments) (function . arguments))))
```

  The resulting expansion is now a list constructed within the **template**
environment and therefore interpreted as a function call.

  > *** Don't use macro "rest" arguments as an implicit function call. Use a
template with an explicit `(function . arguments)` element.

  ## Multiple Patterns

  `syntax-rules` allows for an arbitrary number of pattern/template pairs. A
match is attempted against the first pattern; if it fails, the next is examined.
If no pattern matches, an error is raised.

  ## Syntax Errors

  To produce helpful error messages during expansion:

```scheme
(define-syntax syntax-error
  (syntax-rules ()
    ((syntax-error) (syntax-error "Bad use of syntax error!"))))
```

  We can then write macros that expand into "calls" to `syntax-error`:

```scheme
(define-syntax prohibit-one-arg
  (syntax-rules ()
    ((prohibit-one-arg function argument)
     (syntax-error
      "Prohibit-one-arg cannot be used with one argument."
      function argument))
    ((prohibit-one-arg function . arguments)
     (function . arguments))))
```

  > *** Write a `syntax-error` macro and write "rejection" patterns by expanding
into a call to it.

  ## "Accidental" Matching

  Symbols in patterns match **anything**, which can cause confusion. A pattern
like `(my-named-let name (binding . more-bindings) . body)` matches
`(my-named-let ((x 22) (y "foo")) ...)` because `name` matches the entire list
of bindings.

  > *** Nested list structure in the pattern will match similar nested list
structure in the form, but symbols in the pattern will match **anything**.

  Protect against this by "guarding" with patterns that match bad syntax:

```scheme
(define-syntax my-named-let
  (syntax-rules ()
    ((my-named-let () . ignore)
     (syntax-error "NAME must not be the empty list."))
    ((my-named-let (car . cdr) . ignore)
     (syntax-error "NAME must be a symbol." (car . cdr)))
    ((my-named-let name bindings form . forms)
     (let name bindings form . forms))))
```

  ## Recursive Expansion

  A macro can expand into a form that embeds a call to itself. This creates a
tail-recursive loop. Recursive expansion always produces a nested result.

```scheme
(define-syntax bind-variables
  (syntax-rules ()
    ((bind-variables () form . forms)
     (begin form . forms))
    ((bind-variables ((variable value0 value1 . more) . more-bindings) form . forms)
     (syntax-error "bind-variables illegal binding" (variable value0 value1 . more)))
    ((bind-variables ((variable value) . more-bindings) form . forms)
     (let ((variable value)) (bind-variables more-bindings form . forms)))
    ((bind-variables ((variable) . more-bindings) form . forms)
     (let ((variable #f)) (bind-variables more-bindings form . forms)))
    ((bind-variables (variable . more-bindings) form . forms)
     (let ((variable #f)) (bind-variables more-bindings form . forms)))
    ((bind-variables bindings form . forms)
     (syntax-error "Bindings must be a list." bindings))))
```

  `BIND-VARIABLES` is like `LET*`, but allows omitting values (binding to `#f`)
and omitting parentheses around variable names.

  ### List Recursion Idiom 1. Macro has a parenthesized list of items in a fixed
position. 2. A pattern matching the empty list `()` precedes other matches (Base
case). 3. One or more patterns have dotted tails in the list position, ordered
from most-specific to most-general. 4. Dotted tail component is passed to the
recursive call.

  A minimal list recursion:
```scheme
(define-syntax mli
  (syntax-rules ()
    ((mli ()) (base-case))
    ((mli (item . remaining)) (f item (mli remaining)))
    ((mli non-list) (syntax-error "not a list"))))
```

  ***

  At this point, things get complicated. We are no longer looking at simple
rewrites; we are performing computation to control the macro engine.

  A macro is a compiler. It takes source code (Scheme with extensions) and
generates object code (Scheme without extensions). The language used to write
these compilers is the pattern/template language of `syntax-rules`.

  The computation model is non-procedural. familiar abstractions like
subroutines and named variables don't exist in a recognizable form. But if we
look carefully, they have just been destructured and re-formed into strange
shapes.

  ## List Iteration

  By processing the leftmost variable first, we can create a list iteration:

```scheme
(define-syntax bind-variables1
  (syntax-rules ()
    ((bind-variables1 () . forms)
     (begin . forms))
    ((bind-variables1 ((variable value) . more-bindings) . forms)
     (bind-variables1 more-bindings (let ((variable value)) . forms)))))
```

  We can pretend **the template form is a tail-recursive function call.**
`syntax-rules` functions as a `COND`. Arguments are unnamed, but patterns allow
us to destructure values into "named" variables.

  ### Example: MULTIPLE-VALUE-SET! Mimicking Common Lisp's
`multiple-value-setq`:

```scheme
(define-syntax multiple-value-set!
  (syntax-rules ()
    ((multiple-value-set! variables values-form)
     (gen-temps-and-sets variables () () values-form))))

(define-syntax gen-temps-and-sets
  (syntax-rules ()
    ((gen-temps-and-sets () temps assignments values-form)
     (emit-cwv-form temps assignments values-form))
    ((gen-temps-and-sets (variable . more) temps assignments values-form)
     (gen-temps-and-sets
        more
       (temp . temps)
       ((set! variable temp) . assignments)
       values-form))))

(define-syntax emit-cwv-form
  (syntax-rules ()
    ((emit-cwv-form temps assignments values-form)
     (call-with-values (lambda () values-form)
       (lambda temps . assignments)))))
```

  ## The "Mrs. McCave" Problem (Hygiene)

  > "Did I ever tell you that Mrs. McCave / Had twenty-three sons, and she named
them all Dave?" — Dr. Seuss

  In `multiple-value-set!`, all temporary variables are named `TEMP`. However,
the code works because each `TEMP` identifier created in the same expansion step
refers to the **same** syntactic object, while those created in **different**
steps are unique.

  > *** Introduce associated code fragments in a single expansion step. > ***
Introduce duplicated, but unassociated fragments in different expansion steps.

  ## Ellipses (...)

  The `...` operator is postfix and modifies how the previous form is
interpreted. In a pattern, it matches repeated occurrences in the tail of a list
or vector.

  Example pattern: `(foo var #t ((a . b) c) ...)` Matches: `(foo 11 #t ((moe
larry curly) stooges) ((carthago delendum est) cato))`

  In a template, anything suffixed with an ellipses is repeated as many times as
the pattern variable matched.

```scheme
;; Using ellipses to extend lists (analogous to append)
((gen-temps-and-sets (variable . more) (temps ...) (assignments ...) values-form)
 (gen-temps-and-sets
    more
   (temps ... temp)
   (assignments ... (set! variable temp))
   values-form))
```

  > *** Use ellipses to extend lists while retaining order. > *** Use ellipses
to "flatten" output.

  ## Pattern Labeling Trick

  Using strings (which match only if `equal?`) to put "labels" on patterns
within a single auxiliary macro:

```scheme
(define-syntax mvs-aux
  (syntax-rules ()
    ((mvs-aux "entry" variables values-form)
     (mvs-aux "gen-code" variables () () values-form))
    ;; ... etc
    ))
```

  ## Stack-Machine Style

  Macros so far have been linear. For complex macros, we need subroutines. We
can explicitly pass a **macro continuation**—represented by a list of a string
tag and necessary data.

```scheme
(define-syntax return
  (syntax-rules ()
    ((return ((kbefore ...) . kafter) value)
     (kbefore ... value . kafter))
    ((return () value) value)))
```

  By placing the continuation in a fixed spot (the first position), we treat the
macro processor as a stack machine.

  ### The Obligatory Hack: A Scheme Interpreter A small Scheme interpreter
written as a `syntax-rules` macro. It is incredibly slow.

```scheme
;; [Interpreter code block follows as provided in original text]
(define-syntax scheme-eval
  (syntax-rules ()
    ((scheme-eval expression)
     (macro-call ((quote))
       (! (meval expression (! (initial-environment))))))))
;; ... etc
```
