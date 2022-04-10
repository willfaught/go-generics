# Enrich the type system

## Enable bounded parametric polymorphism for structural typing

### Type variables

Add a new type: type variables. Examples:

```go
// Basically identical to identifiers for types
f
foo
B
Bar
bAz99
```

It is an error for a type variable to occur outside a type abstraction. All type expressions must be closed with no free type variables.

It may be that the current type identifiers can serve as type variables, as long as they are bound by a type abstraction.

### Type abstractions

Type abstractions are functions from types to types, types to expressions, and types to declarations.

Add a new type: type abstractions.

Examples:

```go
// A, B, C, D are any type variables
// T is any type
// Y, Z are any interface types
func(type A) T
func(type A, B) T
func(type A Z) T
func(type A, B Z, C, D Y) T
```

A type abstraction consumes one or more types as arguments and produces an expression as a result.

Add a new expression: type abstractions.

Examples:

```go
// A, B, C, D are any type variables
// E is any expression
// Y, Z are any interface types
func(type A) E
func(type A, B) E
func(type A Z) E
func(type A, B Z, C, D Y) E
```

### Abstractions from types to declarations

Add a new declaration: type abstractions. Examples:

```go
// A, B, C, D are any type variables
// T is any type
// Y, Z are any interface types
func(type A) T
func(type A, B) T
func(type A Z) T
func(type A, B Z, C, D Y) T
```

### Type applications

Add a new type: type applications. Examples:

```go
// A, B, C are any types
A(B)
A(B, C)
```

### Type kinds



# Overview

Constraints:

- Must be Go-like
    - Language features should be orthogonal to each other

High-level strategy:

- Enable bounded parametric polymorphism for structural typing
- Enable interfaces to optionally represent method receivers
- Add methods to predeclared types (like `int`) that represent their operations (like `Add`)

Low-level tactics:

- Add type variables, abstractions, and applications
- Add syntactic sugar for type abstractions and applications
- Convert predeclared types into pseudo-declared types
- Add methods representing operations to pseudo-declared types
- Extend interfaces to be able to represent receivers

Possible further areas of interest:

- Change some existing types to be more tractable as generic types
    - Make function types tractable generic types
- Make exclusion for interfaces not depend on not exporting a method

## Add type variables, type abstractions, and type applications

Add three new types: type variables, type abstractions, and type applications.

A type variable looks like a normal identifier. For example: `T`, `foo`, `Bar123`.

A type abstraction has one or more type variables as parameters, each of which are constrained to implement one interface, and a body type. It does not contain any free/unbound type variables unless they are bound by an outer type function. Should some or all of the params be required to be free in the body? If any type function remains after type calls are evaluated, that's a kind error.

## Convert predeclared types into pseudo-declared types

Move types associated with predeclared identifiers in the universe block (e.g. `bool`, `error`, `int`, `string`) to a `builtin` package (or `core` or whatever you want to call it) and make them exported. Change those predeclared identifiers to be aliases for their counterparts in the `builtin` package. This implies that all packages now dot-import the `builtin` package.

Make this overrideable with an explicit import statement for altering the package qualifier. `import "builtin"` and `import "foo" "builtin"` use the qualifiers `builtin` and `foo`, respectively.



Add methods to built-in types for the built-in operations we want to abstract generically:

func (int) Add(other int) int
func (string)

---

func Const (type a, b) (x a, y b) a { return x }
func Const(x type a, y type b) a { return x }
func Const(x type a interface{}, y type b interface{}) a { return x }

---

Types:

func(type A) Type
func(type A Z) Type
func(type A, B) Type
func(type A, B Z) Type
func(type A, B Z, C, D Y) Type
A(B)
A(B, C)

func(type A) struct { A; Foo *A } // A embeds as interface{}!
func(type K, V) map[K]V
func(type A, B) func(List(A), func(A) B) List(B)
type List func(type T) []T
type Func func(type A, B) func(A) B

// TODO: What is rank-n parametric polymorphism? Google it, Wikipedia isn't helpful

Expressions:

func(type A) Expression
func(type A Z) Expression
func(type A, B) Expression
func(type A, B Z) Expression
func(type A, B Z, C, D Y) Expression
A(B)
A(B, C)

var f func(type S) func(S) S = func(type T) func(t T) T { return t }
var g func(int) int = f(int)

Declarations:

func(type A) Declaration
func(type A Z) Declaration
func(type A, B) Declaration
func(type A, B Z) Declaration
func(type A, B Z, C, D Y) Declaration
A(B)
A(B, C)

func(type T) func MakeSlice(n int) []T // Erases as []interface for the impl! But still typechecks as []int if applied to int, for example
func MakeSlice (n int) []$T

func(type A, B) func Map(f Func(A, B), as List(A)) List(B)
func Map(f Func($A, $B), as List($A)) List($B)
Map(int, string)(converter, xs)

func(type T) func (l *List(T)) Get(index int) (T, bool)
func (l *List($T)) Get(index int) ($T, bool)
List.Get(int)(123)
myList.Get(123)

Examples:

for _, test := range []struct {
    name string
    sort func(type T Orderable) Func(List(T)) List(T)
    xs func(type T Orderable) List(T)
    expected func(type T Orderable) List(T)
} {
    {"test1", insertionSort, oneToTen, oneToTen}
} {

}

---

https://emilymaier.net/words/getting-specific-about-generics/

https://blog.merovius.de/2018/09/05/scrapping_contracts.html

https://go.googlesource.com/proposal/+/master/design/go2draft-generics-overview.md

https://github.com/golang/go/wiki/Go2GenericsFeedback

# Generics proposal

This is a proposal for how to enable [generics](https://en.wikipedia.org/wiki/Parametric_polymorphism) in Go. The need for generics was argued [elsewhere](https://go.googlesource.com/proposal/+/master/design/go2draft-generics-overview.md) by the Go Team, so I won't repeat it here. The design is presented first, followed by remarks.

## Design

The strategy is to extend the type system so interfaces can describe primitive operations, then pre-declare methods for primitive types that implement these interfaces.

Add new types: type variables, type "functions" (abstractions), and type "calls" (applications)

A type variable is a normal type name/identifier. It must at least once be bound by a type func.

A type function has one or more type names as parameters, each of which are constrained to implement one interface, and a body type. It does not contain any free/unbound type variables unless they are bound by an outer type function. Should some or all of the params be required to be free in the body? If any type function remains after type calls are evaluated, that's a kind error.

func(type TypeVariable Type) Type
func(type TypeVariable Type) Expression
func(type TypeVariable Type) Declaration ???

A type call matches all the type params in a type whose underlying type is a type func with the same number of type arguments. Its meaning is the textual substitution of the arguments with the parameters where the parameters occur free/unbound within the type func's body. They are "evaluated" before applying other semantics to them.

Extend interfaces to allow "self" receiver type variables (pointer and non-pointer variants). Allow paren and paren-with-param syntax to allow doc to refer to names like for results.

func F(type A interface{})(g func(A) A) func(A) A {
    return g
}

func I(x int) int {
    return x
}

F(int)(I)
F(int)(I)(123)

func Map(type A interface{})(f func(A) A, xs []A) []A {

}

type T func(type A) struct {
    Name string
    Val []A
    Count int
}

(func (type A interface{})(g func(A) A) func(A) A { return g })(int)(I)

func(type T interface{}) (v *Vector(T)) PushAll(s Stack(T)) (int, error)

type BytePushAller interface {
    PushAll(s Stack(byte)) (int, error)
}

var v Vector(byte)
var p BytePushAller = v

func(type T interface{}) (v *Vector(T)) Read(b []T) (int, error)

var v Vector(byte)
var r io.Reader = v

### Type abstractions

A new type expression, a **type function**, is added:

```go
func(type A B) C
```

Above, `A` is a **type variable**, and `T` is any type. The meaning is similar to value abstractions (Go functions)

```go
func(type A) T

func(type A, B) T

func(type A Z) T

func(type A, B Z, C, D Y) T

T(A)

T(A, B)
```

Types now have kinds. All types in the current spec have kind *type*. All parametric types must have kind *type* before other aspects of semantics are applied to them. I.e. all abstractions must be applied.

Self interfaces:

```go
func(type T) interface { T M() }

func(type T) interface { *T M() }

func(type T) interface { (t1 T) M(t2 T) (t3 T) }

func(type T) interface { (t1 *T) M(t2 T) (t3 T) }

func(type T) struct {
    first, last *T
    all []*T
}

func(type A) T

interface { String() string }

interface { Reset() }

func(type T) interface { *T Reset() }

func(type T) interface { Add(x, y T) T }

func(type T) interface { Add(x *T, y T) }

func(type T) interface { T Add(x T) T }

func(type T) interface { Add(x *T) }

func(type T) interface { *T Add(x T) }
```

Operation methods:

```go
type Add func(type T) interface { T Add(T) T }
type Divide func(type T) interface { T Divide(T) T }
type Multiply func(type T) interface { T Multiply(T) T }
type Negate func(type T) interface { T Negate() T }
type Remainder func(type T) interface { T Remainder(T) T }
type Subtract func(type T) interface { T Subtract(T) T }

type Number func(type T) interface {
    Add(T)
    Divide(T)
    Multiply(T)
    Negate(T)
    Remainder(T)
    Subtract(T)
}

type Equal func(type T) interface { T Equal(T) bool }

type Equatable func(type T) interface {
    Equal(T)
}

type Greater func(type T) interface { T Greater(T) bool }
type GreaterEqual func(type T) interface { T GreaterEqual(T) bool }
type Less func(type T) interface { T Less(T) bool }
type LessEqual func(type T) interface { T LessEqual(T) bool }

type Orderable func(type T) interface {
    Greater(T)
    GreaterEqual(T)
    Less(T)
    LessEqual(T)
}

type And func(type T) interface { T And(T) T }
type Or func(type T) interface { T Or(T) T }
type Not func(type T) interface { T Not() T }

type Boolean func(type T) interface {
    And(T)
    Or(T)
    Not(T)
}

type AndNot func(type T) interface { T AndNot(T) T }
type BitAnd func(type T) interface { T BitAnd(T) T }
type BitOr func(type T) interface { T BitOr(T) T }
type ShiftLeft func(type T) interface { T ShiftLeft(int) T }
type ShiftRight func(type T) interface { T ShiftRight(int) T }
type Xor func(type T) interface { T Xor(T) T }

type Integer func(type T) interface {
    AndNot
    BitAnd
    BitOr
    Number(T)
    ShiftLeft
    ShiftRight
    Xor
}

type Ref func(type T) func(type T) interface { *T Ref() *T }
type Deref func(type T) func(type T) interface { *T Deref() T }

type Pointer func(type T) interface {
    Ref
    Deref
}

type Send func(type E) interface { Send(E) }
type Receive func(type E) interface { Receive() E }

type Channel func(type T) interface {
    Send
    Receive
}

type Function func(type A, R) interface { Call([]A) []R }

type Cap interface { Cap() int }
type Delete func(type I) interface { Delete(I) }
type Index func(type I, E) interface { Index(I) E }
type IndexContains func(type I, E) interface { IndexContains(I) (E, bool) }
type Len interface { Len() int }
type Slice func(type E) interface { Slice(low, high, max int) []E }

type Map func(type I, E) interface {
    Cap
    Delete
    Index(I, E)
    IndexContains(I, E)
    Len
}

type Array func(type E) interface {
    Cap
    Index(int, E)
    Len
    Slice(E)
}

type Slice Array
```

```go
func (x int) Add(y int) int { return x + y }
```

## Remarks

TODO
