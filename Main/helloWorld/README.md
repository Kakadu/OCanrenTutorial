# Say "Hello World!" in OCanren

We first execute the program, and then we read and explain through the source code line by line.
##  Executing the Program

The source code has to be compiled and linked, for which you would need the [Makefile](Makefile).
Now open terminal under your local copy of the [`helloWorld`](./) direcory, and:
```
make
```
This would produce a native-code file `hello.opt`, execution of which by:
```
./hello.opt
```
shall print `hello world!` in your terminal.

** It is probably worth to have two versions of build system: Makefile and dune (which is a de facto standart for OCaml right now) **

## Reading the Program

The first line:
```ocaml
open OCanren
```
makes the names from the module OCanren available for later use.

### Important Interfaces

The source code of the module OCanren resides in
[OCanren.ml](../../Installation/ocanren/src/OCanren.ml)
(It does not have an accompanying `.mli` file).
Inspecting the content thereof,  we shall see that basically
it includes two modules [Logic](../../Installation/ocanren/src/core/Logic.mli)
and [Core](../../Installation/ocanren/src/core/Core.mli), and renames the module
[RStream](../../Installation/ocanren/src/core/RStream.mli)
to Stream, and finally defines the Std module. You should
open the module OCanren at the beginning of every source file where
you use OCanren features.

The second line:
```ocaml
let str = !!("hello world!\n")
```
associates the value name `str` with the expression that
is the prefix operator `!!` (named _primitive injection_)
applied to the string literal `"hello world!\n"`.

### Injecting (or Type Casting) to the OCanren Internal Representation

The operator `!!`, provided by the module Logic, makes type conversion to the OCanren
internal representation, something like what a calculator program
does when it receives an input string "1" and converts it to the integer
or floating-point type for further processing. We could see from the interface that:
```ocaml
val (!!) : 'a -> ('a, 'a logic) injected
```
The `injected` type constructor is provided by the module Logic as an abstract type
 (so we do not concern ourselves with its implementation).

** Maybe it is worth saying here why two type parameters and what they do reperesent **

#### Logic Variables

The `logic` type constructor which appears in the type of `!!` above is also
provided by the module `OCanren.Logic`.
It takes one type parameter and its type representation is exposed: we could
see from the module interface that it has two constructors
`Var` and `Value`,  representing respectively a _logic variable_ and a _concrete value_
over/of the parameter type, in the sense that wrt. the arithmetic expression `1 + x` we
say that `x` is a logic variable over the integer type and `1` is a concrete value of
the integer type.

### Making a Query

The 3rd line:
```ocaml
let _ =
  List.iter print_string @@
    Stream.take ~n:1 @@
      run q (fun q -> ocanren { q == str }) project;;
```
is divided into three sub-expressions by the right associative infix
operator `@@` that is provided by OCaml's core library
[Stdlib](http://caml.inria.fr/pub/docs/manual-ocaml/libref/Stdlib.html). The most
 important sub-expression is:
 ```ocaml
 run q (fun q -> ocanren { q == str }) project
 ```
whose most important part is: `q == str`.

Next we start with explaining the inner most and the most important part: `q == str`,
followed by its immediate enclosing environment which is the _run_ statement:
```ocaml
run q (fun q -> ocanren { q == str }) project
```
and finally the top most expression for taking and printing answers.

** I think it is a right thing to say that there are a few syntaxes for OCanren and there we are using one of them **

### Syntactic Identity and  Unification

Syntactic identity between two expressions _expr_<sub>1</sub> and _expr_<sub>2</sub>
(of the same type) is  denoted _expr_<sub>1</sub> `==` _expr_<sub>2</sub>. Usually we
are given two different expressions both of which  have zero or more sub-expressions
considered as logic variables, and we are interested in finding (type sensitive)
substitutes for these logic variables so that the resulting expressions are
syntactically identical. Finding such substitutes is know as _unification_ of the
original expressions.

**Example** In order for both `(x + 1) == (2 + y)` and `Node (x,1) == Node (2,y)`
to be true, we  replace `x` by `2`and `y` by `1`,  making both sides of `==` the
expression `2 + 1` or `Node (2,1)` respectively. We now unified `( x + 1)` with
`(2 + y)`. Moreover, `Node (x,1)` is unified with `Node (2,y)`.

**Example** How to substitute the logic variable  `z` so that `z == "hello world!"` ?
Trivial:  replacing  `z` with the constant `"hello world!"`. This is essentially what
our program does: solving a unification problem.

### The OCanren Top Level: the _run_ statement

We can parse the `run ...` expression following the syntax below,
 which is given in [EBNF](https://github.com/YueLiPicasso/language-grammars)
except that occurrences of the meta-identifier `etc` signifies omission:
there is no single syntactic category named `etc`.

```ebnf
top level = 'run',  size indicator, goal, answer handler ;

size indicator =  'one' | 'two' | 'three' | 'four' | 'five'
                | 'q'   | 'qr'  | 'qrs'   | 'qrst' | 'qrstu'
		| '(', size indicator, ')'
		| 'succ', size indicator ;

goal = 'fun', parameters, '->', goal body ;

parameters = etc ;

goal body = 'ocanren', '{', pretty goal body, '}' ;

pretty goal body = etc ;

answer handler = 'project' | etc ;
```
A goal asks: what values shall be assumed by the parameters
so that the proposition as given by the goal body (in which these parameters are expected to occur) holds?

 The `ocanren { }` environment in a goal body instructs the  [camlp5](https://camlp5.github.io/)
 preprocessor to transform on the syntactic level the `pretty goal body` into the more
 verbose and less intuitive OCanren internal representation. As a reference, the rules
 by which the preprocessing is done is given in `pa_ocanren.ml`
 [where](../../Installation/ocanren/camlp5/pa_ocanren.ml#L238) we could find,
 for example, the `==` symbol is transformed into the name `unify`.

The number of parameters shall
agree with the size indicator, where `q`...`qrstu` are just alternative names for
`one`...`five` respectively. If there are more than five parameters, the successor function
`succ` can be applied to build larger size indicators, e.g., `(succ (succ five))` is for seven
parameters.

The answer handler is a type converter from the OCanren
internal representation to a user-level representation. When there is no free logic variables in
the answer we use `project`. The `Not_a_value` exeption (provided by OCanren.Logic) is thrown if
we use `project` as the handler but the answer contains free logic variables: in this case
some other handler shall be used.

The `run` function and the size indicators are provided by Core.
Basic answer handlers are provided by Logic.

### Taking and Displaying Answers


The top level constructs a lazy stream out of which an arbitrary number of
 answers could be pulled, subject to answer availability.
We use a stream instead of a finite list to
 hold the answers because generally the set of all answers is enumerable.
Omission of the optional argument `~n` of `take` means "take all". Finally we use OCaml
 standard functions to iterate through the taken list of answers and print each one in
 the terminal. Our program has only one answer.
