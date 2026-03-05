% Extended Example: Regular expressions
% Martin Sulzmann


# Regular expression derivatives 

We show how to mimic pattern matching over algebraic data types in Go.


## Some theory on regular expressions first

### Syntax

In EBNF Syntax:


~~~~~~~~~~~~~~~

r,s ::= x | y | z | ...         Symbols aka letters taken from a finite alphabet
    |  epsilon                  Empty word
    |  phi                      Empty language
	|  r + r                    Alternatives
	|  r . r                    Sequence aka concatenation
	|  r*                       Kleene star

u,v,w ::= epsilon               empty word
      |  w . w                  concatenation
~~~~~~~~~~~~~~~~~~~


Sigma denotes the finite set of alphabet symbols.

Regular expressions are popular formalism to specify (infinitely many) patterns of input.

For example,

~~~~~~~~~~
 (open . (read + write)* . close)*
~~~~~~~~~~~~~~~~~

specifies the valid access patterns of a resource usage policy.

We assume that `open`, `read`, `write`, `close` are the primitive events (symbols) which will be
recorded during a program run.


### Membership

`L(r)` denotes the set of words represented by the regular expression r.

Standard (denotational) formulation of L(r):

~~~~~~~~~~
L(x) = { x }

L(epsilon) = { epsilon }

L(phi) = { }

L(r + s) = { w | w in L(r) or w in L(s) }

L(r . s) = { v . w | v in L(r) and w in L(s) }

L(r*) = { epsilon } cap { w_1 . ... . w_n | w_1, ..., w_n in L(r) and n >=1 }
~~~~~~~~~~~~~~~


#### Membership Test


We ask the question. Given a regular expression r and a word w does it hold that w is an element in L(r)?
The classical approach is to turn the regular expression into an automata
(for example via the Thompson NFA construction).

For example, consider the regular expression

~~~~
a* . b . c*
~~~~~~

that denotes the language of an arbitrary sequence of a's followed by a b,
followed by an arbitrary sequence of c's.
We do not strictly follow the Thompson method, rather we highlight
the principle of turning regular expressions into automata.

Here is a possible automata.

~~~~~
1 --a--> 1
1 --b--> 2
2 --c--> 2

initial: 1
final: 2
~~~~~~

There are two states, 1 and 2.
State 1 is the initial state and state 2 is the final state.
A transitition such as `1 --a--> 1` states if we are in state 1,
we can consume the input symbol a and then reach state 1 (so we effectively loop
for a's).
If the input symbol is b, we reach state 2.
What if the input symbol is c? There does not seem to be transition
labeled with c.
A common assumption is to omit transitions that lead to a stuck state.

We complete the above automata by adding all such transitions.

~~~~~~
1 --c--> 3
2 --a--> 3
2 --b--> 3
3 --a--> 3
3 --b--> 3
3 --c--> 3
~~~~~~~

If in state 1 and the input symbol is c, we reach state 3.
State 3 is just some state (not final) and effectively represents
the error state (the input word is not part of the language).


To summarize. The purpose of an automata is to
consume symbols and executing transitions.
In each step, we consume an input symbol and apply one of the transitions.

For example, consider input "ccba".

~~~~
1 --c--> 1 --c--> 1 --b--> 2 --a--> 2
~~~~~

State 2 is final. Hence, "ccba" is word of the language.

As another example, consider input "ccbb".

~~~~
1 --c--> 1 --c--> 1 --b--> 2 --b--> 3
~~~~~

State 3 is not final. So, "ccbb" is not a word of the language.

#### Membership Test via Derivatives


We consider an alternative method to carry out the membership test based on
Brzozowski [https://en.wikipedia.org/wiki/Brzozowski_derivative]().
He introduced a symbolic method to construct an automata from a regular expression
based on the concept of derivatives.

Given some expression r and a symbol x, we obtain the *derivative*
of r w.r.t. x, written d(r,x), by taking way the leading symbol x from r.

In semantic terms, d(r,x) can be described as follows:

~~~~~~~~~~~~~~
L(d(r,x)) = x \ L(r)
~~~~~~~~~~~~~~~~~

* x \ L(r) denotes the left quotient, i.e. the language `{ w | x . w in L(r)}`.

    * We write `.` to denote concatenation. In some exposition this is left silent, i.e. `x w`.

* Hence, the derivative `d(r,x)` denotes the set of all words from L(R) where the leading symbol x has been removed.


Thus, it is easy to solve the word problem.
Let `w` be a word consisting of symbols `x1 . x2 .... xn-1 . xn`.

Compute

~~~~~~~~~~~~~
d(r,x1) = r1
d(r1,x2) = r2
...
d(rn-1,xn) = rn
~~~~~~~~~~~~~~~~~~~

That is, we repeatidely build the derivative of r w.r.t symbols xi.

Check if the final expression `rn` is nullable.
An expression s is *nullable* if epsilon in L(s).


### Formalizing Nullability and the Derivative operation

It is surprisingly simply to decide nullability by observing
the structure of regular expression.

We write `n(r)` to denote the nullability test which yields a Boolean
value (true/false).

~~~~~~~
n(x) where xi is a symbol never holds.

n(epsilon) always holds.

n(phi) never holds.

n(r + s) holds iff n(r) holds or n(s) holds.

n(r . s) holds iff n(r) holds and n(s) holds.

n(r*) always holds.
~~~~~~~~~

A similar approach (definition by structural recursion) works
for the derivative operation.

We write `d(r,x)` to denote the derivative obtained by extracting the leading symbol `x` from expression `r`. For each derivative, we wish that the following holds:
`L(d(r,x)) = x \ L(r)`.


As in case of the nullability test, the derivative operation is defined by observing
the structure of regular expression patterns. Instead of a Boolean value, we yield
another expression (the derivative).

~~~~~~~~~
d(x,y) =   either epsilon if x == y or phi otherwise

d(epsilon,y) = phi

d(phi,y)     = phi

d(r + s, y)  = d(r,y) + d(s,y)

d(r . s, y)  =  if n(r)
                then d(r,y) . s +  d(s,y)
                else d(r,y) . s

d(r*, y)     = d(r,y) . r*
~~~~~~~~~

## Examples

Let's consider some examples to understand the workings of the derivative
and nullability function.

We write `r -x-> d(r,x)` to denote application of the derivative
on some expression `r` with respect to symbol `x`.

~~~~~~~~~
       x*
-x->   d(x*,x)
       = d(x,x) . x*       
       = epsilon . x*      

-x->   epsilon . x*
       = d(epsilon,x) . x* + d(x*,x)     -- n(epsilon) yields true
       = phi . x* + epsilon . x*
~~~~~~~~~~

So repeated applicaton of the derivative on `x*$ for input string "x.x"
yields `phi . x* + epsilon . x*`.
Let's carry out the nullability function on the final expression.

~~~~~~~~~
    n(phi . x* + epsilon . x*)
    = n(phi .x*) or n(epsilon . x*)
	= (n(phi) and n(x*)) or (n(epsilon) and n(x*))
	= (false and true) or (true and true)
	= false or true
	= true
~~~~~~~~~~

The final expression `phi . x* + epsilon . x*` is nullable.
Hence, we can argue that expression `x*` matches the input word "x.x".



## Implementation in Go


We make use of (extensible) interfaces to mimic pattern matching in Go.

~~~~~~~~~~{.go}
package main

import "fmt"

type RE interface {
	deriv(byte) RE
	nullable() bool
}

type Eps int
type Phi int
type Letter byte
type Kleene [1]RE
type Alt [2]RE
type Seq [2]RE

// test if regex is nullable
func (r Eps) nullable() bool {
	return true
}
func (r Phi) nullable() bool {
	return false
}
func (r Letter) nullable() bool {
	return false
}

func (r Kleene) nullable() bool {
	return true
}
func (r Alt) nullable() bool {
	return r[0].nullable() || r[1].nullable()
}
func (r Seq) nullable() bool {
	return r[0].nullable() && r[1].nullable()
}

// build the derivative wrt x
func (r Eps) deriv(x byte) RE {
	return Phi(1)
}
func (r Phi) deriv(x byte) RE {
	return Phi(1)
}

func (r Letter) deriv(x byte) RE {
	y := (byte)(r)
	if x == y {
		return Eps(1)
	} else {
		return Phi(1)
	}
}

func (r Kleene) deriv(x byte) RE {
	return (Seq)([2]RE{r[0].deriv(x), r})
}

func (r Alt) deriv(x byte) RE {
	return (Alt)([2]RE{r[0].deriv(x), r[1].deriv(x)})
}

func (r Seq) deriv(x byte) RE {
	if r[0].nullable() {
		return (Alt)([2]RE{(Seq)([2]RE{r[0].deriv(x), r[1]}), r[1].deriv(x)})
	} else {
		return (Seq)([2]RE{r[0].deriv(x), r[1]})
	}
}

func testRegEx() {
	var c Letter = Letter('c')
	eps := (Eps)(1)
	phi := (Phi)(1)
	r1 := (Alt)([2]RE{eps, phi})
	r2 := (Seq)([2]RE{c, r1})

	fmt.Printf("%b \n", (r2.deriv('c')).nullable())
}

func main() {

	testRegEx()
}
~~~~~~~~~~~~~~~

## Summary

Each pattern matching function is method of the `RE` inferface.

Each case of the description of regular expression is represented
by a specific type.

The methods of the `RE` inferface provide an implementation for each case.




# Some (further) regex exercises

* Parse the term “aa” with the regex a 
* Parse the term “b” with the regex a + b
* Parse the term “c” with the regex a + b
* Parse the term “ab” with the regex a . b 
* Parse the term “ab” with the regex (a + b) . (a + b)
* Parse the term "aa" with the regex a*
* Parse the term “abb” with the regex a.b*
* Parse the term “acd” with the regex a . (b + c)* . d

## Solutions

### By hand calculations

* Parse the term “aa” with the regex a 

~~~~~~~~~~~
      a
-a->  d(a, a)
      = epsilon
-a->  d(epsilon, a)
      = phi
        
check nullability

      n(phi)
      = false
~~~~~~~~~~~

* Parse the term “b” with the regex a + b

~~~~~~~~~~~
      a + b
-b->  d(a + b, b)
      = phi + epsilon
        
check nullability

      n(phi + epsilon)
      = n(phi) or n(epsilon)
      = false or true
      = true
~~~~~~~~~~~

* Parse the term “c” with the regex a + b

~~~~~~~~~~~
      a + b
-c->  d(a + b, c)
      = phi + phi
        
check nullability

      n(phi + phi)
      = n(phi) or n(phi)
      = false or false
      = false
~~~~~~~~~~~

* Parse the term “ab” with the regex a . b 

~~~~~~~~~~~
      a . b
-a->  d(a . b, a)         -- n(a) yields false
      = d(a, a) . b
      = epsilon . b
-b->  d(epsilon . b, b)   -- n(epsilon) yields true
      = d(epsilon, b) . b + d(b, b)
      = (phi . b) + epsilon
        
check nullability

      n((phi . b) + epsilon)
      = n(phi . b) or n(epsilon)
      = (n(phi) and n(b)) or n(epsilon)
      = false or true
      = true
~~~~~~~~~~~

* Parse the term “ab” with the regex (a + b) . (a + b)

~~~~~~~~~~~
        (a + b) . (a + b)
-a->    d((a + b) . (a + b), a)   -- n(a + b) yields false
        = d(a + b, a) . (a + b)
        = (epsilon + phi) . (a + b)
-b->    d((epsilon + phi) . (a + b), b)   -- n(epsilon + phi) yields true
        = (d(epsilon + phi, b) . (a + b)) + d(a + b, b)
        = ((phi + phi) . (a + b)) + (phi + epsilon)
        
check nullability

        n(((phi + phi) . (a + b)) + (phi + epsilon))
        = n((phi + phi) . (a + b)) or n(phi + epsilon)
        = (n(phi or phi) and n(a or b)) or n(phi + epsilon)
        = (false and false) or true
        = false or true
        = true

~~~~~~~~~~~

* * Parse the term "aa" with the regex a*

~~~~~~~~
       a*
-a->   d(a*,a)
       = d(a,a) . a*
       = epsilon . a*
-a->   d(epsilon .a*, a)
       = d(epsilon, a).a* + d(a*,a)        -- n(epsilon) yields true
       = phi.a* + epsilon.a*

We assume that * binds tigher than . which binds tigher than +.
So, phi.a* is interpreted as phi . (a*) adding some explicit parantheses.
~~~~~~~~~~~

 
* Parse the term “abb” with the regex a.b*

~~~~~~~~~~~
        a . b*
-a->    d(a . b*, a)        -- n(a) yields false
        = d(a, a) . b*
        = epsilon . b*
-b->    d(epsilon . b*, b)   -- n(epsilon) yields true
        = (d(epsilon, b) . b*) + d(b*, b)
        = (phi . b*) + (epsilon . b*)
-b->    d((phi . b*) + (epsilon . b*), b)
        = d(phi . b*, b) + d(epsilon . b*, b)  -- left side n(phi) yields false, right side n(epsilon) yields true
        = (phi . b*) + ((phi . b*) + (epsilon . b*))
        
check nullability

        n((phi . b*) + ((phi . b*) + (epsilon . b*)))
        = n(phi . b*) or (n(phi . b*) or n(epsilon . b*))
        = false or (false or true)
        = false or true
        = true
~~~~~~~~~~~

* Parse the term “acd” with the regex a . (b + c)* . d

~~~~~~~~~~~
        a . (b + c)* . d
-a->    d(a . (b + c)* . d, a)        -- n(a) yields false
        = d(a, a) . ((b + c)* . d)
        = epsilon . ((b + c)* . d)
-c->    d(epsilon . ((b + c)* . d), b) -- n(epsilon) yields true
        d(epsilon, b) . ((b + c)* . d) + d((b + c)* . d), b)
        phi . ((b + c)* . d)) + ((((eps + phi) . (b + c)*) . d) + phi)
-d->    d(phi . ((b + c)* . d)) + ((((phi + epsilon) . (b + c)*) . d) + phi), c)
        = d(phi . ((b + c)* . d)), c) + d((((phi + epsilon) . (b + c)*) . d) + phi, c)
        = phi . ((b + c)* . d)) + (((((phi + phi) . (b + c)*) + ((phi + phi) . (b + c)*)) . d) + eps) + phi

check nullability

        n(phi . ((b + c)* . d)) + n(((((phi + phi) . (b + c)*) + ((phi + phi) . (b + c)*)) . d) + eps) + phi)
        = n(phi . ((b + c)* . d))) or n((((((phi + phi) . (b + c)*) + ((phi + phi) . (b + c)*)) . d) + eps) + phi)
        = (n(n(phi) and (n((b + c)*) and n(d)))) or n(((((n(phi + phi) and n((b + c)*)) or (n(phi + phi) and n((b + c)*))) and n(d)) or n(eps)) or n(phi))
        = (fales and (true and false)) or (((((false and true) or (false and true)) and false) or true) or false)
        = (false and false) or (((((false) or (false)) and false) or true) or false)
        = false or (((false and false) or true) or false)
        = false or ((false or true) or false)
        = false or (true or false)
        = false or true
        = true
~~~~~~~~~~~

### By running through the implementation.

*We use the pretty-print extension discussed in the upcoming exercise*

Running the below

~~~~~{.go}
	// (a+b)*
	a := Letter ('a')
        b := Letter ('b')
        r := (Kleene)([1]RE{(Alt)([]RE{a,b})})


	fmt.Printf("r = %s \n", r.pretty())

        // match r against "ab"

        // match a
	r1 := r.deriv('a')
        fmt.Printf("deriv(r,a) = %s \n", r1.pretty())

        // match b
        r2 := r1.deriv('b')
        fmt.Printf("deriv(deriv(r,a),b) = %s \n", r2.pretty())

        // nullability check
        fmt.Printf("%b \n", r2.nullable())       
~~~~~~~~~~~

yields

~~~~~~
r = ((a+b))* 
deriv(r,a) = ((eps+phi).((a+b))*) 
deriv(deriv(r,a),b) = (((phi+phi).((a+b))*)+((phi+eps).((a+b))*)) 
%!b(bool=true) 
~~~~~~~~~


# Some (further) exercises 

## Matching with derivatives (simple)

Based on the `nullable` and `deriv` method it is easy to write
a `matcher` function to check if a regular expression matches
a string.

1. We repeatedly build derivatives.

2. Once the string is empty, we check if the resulting regular expression is nullable.

Implement such a `matcher` function in Go.

## Simplifications (advanced)

If we repeatedly build the derivative, the size of the resulting terms is growing.

For example, consider the following example where
we write write `r --x--> s` for `s = d(r,x)`.

~~~~~~~~~~~
       x*
--x--> d(x,x) . x*
       = epsilon . x*
--x--> d(epsilon . x*, x)
       = d(epsilon,x) . x* + d(x*,x)
       = phi . x* + epsilon . x*
--x--> d(phi . x* + epsilon . x*, x)
       = d(phi . x*, x) + d(epsilon . x*, x)
       = phi . x* + phi . x* + epsilon . x*
~~~~~~~~~~~

and so on.

It is desirable to keep the size of the regular expressions small.

[Brzozowski](https://en.wikipedia.org/wiki/Brzozowski_derivative) observed that few (syntactic) simplification rules are sufficient to ensure
that the size of derivatives will not grow.

~~~~~~~~~~~~~~~
(Idempotence)  r + r = r

(Associativity) (r + s) + t = r + (s + t)

(Commutativity) r + s = s + r
~~~~~~~~~~~~~~~~

The (Idempotence) rule says in case of  syntactically equal elements in an alternative,
we can drop one of the elements.

The (Associativity) and (Commutativity) rule are for convenience so that we can
move (syntactically) equal elements next to each other.

For example, consider

~~~~~~~~
           (r + s) + r
=_Comm     r + (r + s)
=_Assoc    (r + r) + s
=_Idemp    r + s
~~~~~~~~~~


### Hints

There is no need to perform simplifications under a Kleene star.

The main challenge is to re-order alternative elements such that
we can apply the (Idempotence) rule. For example, consider

~~~~~~
           (r + s) + (t + r)
=_Comm     (r + s) + (r + t)
=_Comm     (s + r) + (r + t)
=_Assoc    s + (r + (r + t))
=_Assoc    s + ((r + t) + t)
=_Idemp    s + (r + t)
~~~~~~~~~

Here is a possible approach how to deal with such cases.

1. Introduce an intermediate representation of regular expressions where alternatives are kept in a list.

2. Transform into this intermediate form.

3. Remove any duplicates into a list of alternatives.

4. Transform the intermediate back into the given format for regular expressions.

## Some sample solutions

### Derivative matcher

~~~~~{.go}
func matcher(r RE, s string) bool {
	var b bool
	if len(s) == 0 {
		b = r.nullable()
	} else {
		r2 := r.deriv(s[0])
		b = matcher(r2, s[1:len(s)])
	}
	return b
	 
}
~~~~~~~~~


### Simplifications

~~~~~~~{.go}
package main

import "fmt"

type RE interface {
	pretty() string
	deriv(byte) RE
	nullable() bool
	testSeq() (RE, RE, bool)
	getAlt() []RE
	rightAssoc() RE
	noDups() RE
	flatten() RE
}

type Eps int
type Phi int
type Letter byte
type Kleene [1]RE
type Alt []RE
type Seq [2]RE

// pretty printer
func (r Eps) pretty() string {
	return "eps"
}

func (r Phi) pretty() string {
	return "phi"
}

func (r Letter) pretty() string {
	b := []byte{(byte)(r)}
	return (string)(b)
}

func (r Alt) pretty() string {
	if len(r) == 0 {
		return ""
	}

	var x string
	x = r[0].pretty()
	for i, e := range r {
		if !(i == 0) {
			x += "+" + e.pretty()
		}
	}
	return ("(" + x + ")")
}

func (r Seq) pretty() string {
	return ("(" + r[0].pretty() + "." + r[1].pretty() + ")")
}

func (r Kleene) pretty() string {
	return ("(" + r[0].pretty() + ")*")
}

// test if regex is nullable
func (r Eps) nullable() bool {
	return true
}
func (r Phi) nullable() bool {
	return false
}
func (r Letter) nullable() bool {
	return false
}

func (r Kleene) nullable() bool {
	return true
}
func (r Alt) nullable() bool {
	for _, x := range r {
		if x.nullable() {
			return true
		}
	}
	return false
}
func (r Seq) nullable() bool {
	return r[0].nullable() && r[1].nullable()
}

// build the derivative wrt x
func (r Eps) deriv(x byte) RE {
	return Phi(1)
}
func (r Phi) deriv(x byte) RE {
	return Phi(1)
}

func (r Letter) deriv(x byte) RE {
	y := (byte)(r)
	if x == y {
		return Eps(1)
	} else {
		return Phi(1)
	}
}

func (r Kleene) deriv(x byte) RE {
	return (Seq)([2]RE{r[0].deriv(x), r})
}

func (r Alt) deriv(x byte) RE {
	rs := []RE{}
	for _, e := range r {
		rs = append(rs, e.deriv(x))
	}
	return (Alt)(rs)
}

func (r Seq) deriv(x byte) RE {
	if r[0].nullable() {
		return (Alt)([]RE{(Seq)([2]RE{r[0].deriv(x), r[1]}), r[1].deriv(x)})
	} else {
		return (Seq)([2]RE{r[0].deriv(x), r[1]})
	}
}

// testSeq, check for seq and if yes return left and right operand
func (r Phi) testSeq() (RE, RE, bool) {
	return (Eps)(1), (Eps)(1), false
}

func (r Eps) testSeq() (RE, RE, bool) {
	return (Eps)(1), (Eps)(1), false
}

func (r Letter) testSeq() (RE, RE, bool) {
	return (Eps)(1), (Eps)(1), false
}

func (r Kleene) testSeq() (RE, RE, bool) {
	return (Eps)(1), (Eps)(1), false
}

func (r Alt) testSeq() (RE, RE, bool) {
	return (Eps)(1), (Eps)(1), false
}

func (r Seq) testSeq() (RE, RE, bool) {
	return r[0], r[1], true
}

// getAlt, check for Alt and if yes return list of alternatives,
// otherwise, just return a list with the object on which we called getAlt
func (r Phi) getAlt() []RE {
	return []RE{r}
}

func (r Eps) getAlt() []RE {
	return []RE{r}
}

func (r Letter) getAlt() []RE {
	return []RE{r}
}

func (r Kleene) getAlt() []RE {
	return []RE{r}
}

func (r Alt) getAlt() []RE {
	return r
}

func (r Seq) getAlt() []RE {
	return []RE{r}
}

// right associative normal form
// Seq (Seq (r,s), t) => Seq (r, Seq (s,t))
func (r Eps) rightAssoc() RE {
	return r
}

func (r Phi) rightAssoc() RE {
	return r
}

func (r Letter) rightAssoc() RE {
	return r
}

func (r Kleene) rightAssoc() RE {
	return r
}

func (r Alt) rightAssoc() RE {
	rs := []RE{}
	for _, e := range r {
		rs = append(rs, e.rightAssoc())
	}
	return (Alt)(rs)
}

func (r Seq) rightAssoc() RE {
	var x RE
	left := r[0].rightAssoc()
	right := r[1].rightAssoc()
	l1, l2, b := left.testSeq()
	if b {
		x = (Seq)([2]RE{l1, (Seq)([2]RE{l2, right})})
	} else {
		x = (Seq)([2]RE{left, right})
	}
	return x
}

// flatten alternatives within a list of alternatives
func (r Eps) flatten() RE {
	return r
}

func (r Phi) flatten() RE {
	return r
}

func (r Letter) flatten() RE {
	return r
}

func (r Kleene) flatten() RE {
	return r
}

func (r Seq) flatten() RE {
	return r
}

func (r Alt) flatten() RE {
	xs := []RE{}
	for _, e := range r {
		xs = append(xs, e.flatten().getAlt()...)
	}
	return (Alt)(xs)
}

// We reduce equality test to equality among their pretty printed representation
func eqRE(x RE, y RE) bool {
	return x.pretty() == y.pretty()
}

//check if elem
func elem(x RE, xs []RE) bool {
	for _, e := range xs {
		if eqRE(x, e) {
			return true
		}
	}
	return false
}

//remove duplicates in a slice
func nub(r []RE) []RE {
	curr := []RE{}
	for _, e := range r {
		if !elem(e, curr) {
			curr = append(curr, e)
		}
	}
	return curr
}

//remove duplicate elements in the list of alternatives
func (r Eps) noDups() RE {
	return r
}

func (r Phi) noDups() RE {
	return r
}

func (r Letter) noDups() RE {
	return r
}

func (r Alt) noDups() RE {
	xs := []RE{}
	for _, e := range r {
		xs = append(xs, e.noDups())
	}
	return (Alt)(nub(xs))
}

func (r Seq) noDups() RE {
	return (Seq)([2]RE{r[0].noDups(), r[1].noDups()})
}

func (r Kleene) noDups() RE {
	return r
}

// exhaustive simplification
func simp(r RE) RE {
	var y RE
	x := r.rightAssoc().flatten().noDups()
	if r.pretty() == x.pretty() {
		y = x
	} else {
		y = simp(x)
	}
	return y
}

func testRegEx() {
	var c Letter = Letter('c')
	eps := (Eps)(1)
	phi := (Phi)(1)
	r1 := (Alt)([]RE{eps, phi})
	r2 := (Seq)([2]RE{c, r1})

	fmt.Printf("%b \n", (r2.deriv('c')).nullable())
}

func matcher(r RE, s string) bool {
	var b bool
	if len(s) == 0 {
		b = r.nullable()
	} else {
		r2 := r.deriv(s[0])
		b = matcher(r2, s[1:len(s)])
	}
	return b

}

func testRegEx2() {
	x := Letter('x')
	r := (Seq)([2]RE{(Seq)([2]RE{x, x}), x})

	fmt.Printf("%s \n", simp(r).pretty())
}

func testRegEx3() {
	x := Letter('x')
	r := (Alt)([]RE{(Alt)([]RE{x, x}), x})

	fmt.Printf("%s \n", r.pretty())
}

func testRegEx4() {
	x := Letter('x')
	r := (Kleene)([1]RE{x})

	fmt.Printf("%s \n", r.pretty())

	r1 := simp(r.deriv('x'))

	fmt.Printf("%s \n", r1.pretty())

	r2 := simp(r1.deriv('x'))

	fmt.Printf("%s \n", r2.pretty())

	r3 := simp(r2.deriv('x'))

	fmt.Printf("%s \n", r3.pretty())
}

func main() {

	testRegEx4()
}
~~~~~~~~~~~~~

