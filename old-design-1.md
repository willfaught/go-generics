# Outline

- Intro
- Design
- Upsides
- Downsides
- Remarks
- Related

# Email

I wasn't able to finish this proposal for a design of generics for the Go programming language before another proposal was approved. , and that design will ship soon in Go 1.18. Since the approved generics design proposal will ship soon in Go 1.18, there seems little point in finishing mine, but I thought it might be fun to share what I have so far.

Your thoughts and feedback are most welcome!

# Generics for Go

## Design

TODO: review convertible, assignable, make sure both are used correctly, and assignable is included where needed (it's needed separately where explicit conversion shouldn't be needed)
TODO: in remarks: not including higher-kinded types, to be compatible with maps, and simplify the language. lookup that term to be sure.

### Generics consistency

*Define the equality operations for all types.*

For the types that don't yet have them:

- Functions: Compare the corresponding memory addresses. The time complexity is constant.
- Maps: Compare the corresponding `*runtime.hmap` (pointer) values. The time complexity is constant.
- Slices: Compare the corresponding `runtime.slice` (non-pointer struct) values. The time complexity is constant.

Examples:

```go
// Functions

func F1() {}
func F2() {}

var F3 = F1
var F4 = F2

F1 == F1 && F1 != F2 && F1 == F3 && F1 != F4
F2 == F2 && F2 != F3 && F2 == F4
F3 == F3 && F3 != F4
F4 == F4

// Maps

var M1 = map[int]int{}
var M2 = map[int]int{}

var M3 = M1
var M4 = M2

M1 == M1 && M1 != M2 && M1 == M3 && M1 != M4
M2 == M2 && M2 != M3 && M2 == M4
M3 == M3 && M3 != M4
M4 == M4

// Slices

var S1 = []int{}
var S2 = []int{}

var S3 = S1
var S4 = S2

S1 == S1 && S1 != S2 && S1 == S3 && S1 != S4
S2 == S2 && S2 != S3 && S2 == S4
S3 == S3 && S3 != S4
S4 == S4

S1 = make([]int, 2)

S1 == S1[:] // The lengths, capacities, and pointers are equal
S1 != S1[:1] // The lengths aren't equal
S1[:1] != S1[:1:1] // The capacities aren't equal
S1 != append(S1, 0)[:2:2] // The pointers aren't equal
```

*Define the nil value for all types.*

The predeclared identifier `nil` is now [assignable](https://go.dev/ref/spec#Assignability) to all types, and is the zero ([nil](https://en.wiktionary.org/wiki/nil#Noun)) value for its type. For example, array, boolean, numeric, string, and struct types all now have a nil value, which is their respective zero value.

Examples:

```go
// Assignment

var _ [N]T = nil
var _ bool = nil
var _ int = nil
var _ string = nil
var _ struct{} = nil

// Conversion

var _ = [N]T(nil)
var _ = bool(nil)
var _ = int(nil)
var _ = string(nil)
var _ = struct{}(nil)

// Equality

[N]T{} == [N}T(nil) && [N]T{} == nil
false == bool(nil) && false == nil
0 == int(nil) && 0 == nil
"" == string(nil) && "" == nil
struct{}{} == struct{}(nil) && struct{}{} == nil
```

### Type variables

*Add type variables to types.*

A **type variable** is a placeholder type that is substituted for another type called a **type argument**. It can appear within type, variable, function, and method declarations; method signatures; and function literals.

A type variable is **[constrained](https://en.wikipedia.org/wiki/Parametric_polymorphism#Bounded_parametric_polymorphism)** by an interface, called a **constraint**. Only constraint methods, and operations available for all types, may be used with a type variable value. Type arguments must be assignable to the constraint.

A type variable permits assignment and equality operations with nil and values of identical type variables, method expressions, and method values.

To ensure [parametricity](https://en.wikipedia.org/wiki/Parametricity), a type variable doesn't permit type conversions, type assertions, or type switches. Reflection of a type variable's value only reveals information about the constraint. Type variables are only assignable to identical type variables.

Type variables look like [identifiers](https://golang.org/ref/spec#Identifiers) with a `$` prefix, and cannot be [blank](https://golang.org/ref/spec#Blank_identifier). The case of the first character has no meaning, although type variables are case-sensitive.

Examples:

```go
// Example names

var _ $a
var _ $b
var _ $c
var _ $A
var _ $B
var _ $C
var _ $foo
var _ $BAR
var _ $baz_BOZ

// $A is constrained by C, which has method M, which has an int parameter and result

var x $A // x == nil
x = nil // Valid because x and nil have identical type variables (via assignability)

var _ func(C, int) int = C.M // Method expression
var _ func(int) int = x.M // Method value

x == x // Valid because x and x have identical type variables
x == nil // Valid because x and nil have identical type variables (via assignability)

$A(x) // Invalid because type conversions aren't permitted
MyType(x) // Same as above
interface{}(x) // Same as above
interface{ M(int) int }(x) // Same as above

x.($A) // Invalid because type assertions aren't permitted
x.(MyType) // Same as above
x.(interface{}) // Same as above
x.(interface{ M(int) int }) // Same as above

switch x.(type) {} // Invalid because type switches aren't permitted
```

### Type abstractions

*Add type abstractions to type declarations and signatures.*

A **type abstraction** binds one or more type variables for substitution with type arguments. There are two parts: the **type parameters**, which are a non-empty sequence of mutually-distinct type variables and their associated constraints; and the **body**, which is a type, a function or method declaration, or a function literal, that contains zero or more type variables.

A type variable that isn't a type parameter is **bound** by the closest enclosing type abstraction with an identical type parameter. A type variable that isn't bound is **free**. Code that contains a free type variable is **open**; code that doesn't is **closed**. Declarations in [file blocks](https://golang.org/ref/spec#Blocks) must be closed. A type parameter might not have any corresponding bound type variables in the body. A bound type variable shares the constraint of its corresponding type parameter.

There are two kinds of type abstraction: [values that abstract types](https://en.wikipedia.org/wiki/Parametric_polymorphism), called **type-to-value abstractions** or **generic values**; and [types that abstract types](https://en.wikipedia.org/wiki/Type_operator), called **type-to-type abstractions** or **generic types**.

*Add type-to-value abstractions to values.*

A type-to-value abstraction abstracts the type of the function value specified by function and method declarations, method signatures, and function literals. It's implied by the use of type variables in parameter types. Constraints default to an empty interface if unspecified. Constraints may only be specified for present type variables.

To ensure parametricity, if a type variable appears in a function or method body, or in the result type of a function or method declaration or a function literal, it must also appear within a parameter type of the same function or method, or of an outer function or method within the type abstraction that binds the type variable.

# TODO: Parametricity: allow for returning “free” tyvar if tyapp arg; blob also applies to values with type tyvar, not just variables
# TODO: In tyabs typarams are inferred by tyvars in params
# TODO: func ctor wrapping tycon with no args, NewList() List[T]

Examples:

```go
// C, C1, C2, and so on are constraints
// [...] is a non-empty, unordered set of type parameters and constraints

// Function declarations

func F($A) // $A is constrained by interface{}
func F[$A interface{}]($A) // Same as above

func F($A, $B) // $A and $B are constrained by interface{}
func F[$A interface{}]($A, $B) // Same as above
func F[$B interface{}]($A, $B) // Same as above
func F[$A, $B interface{}]($A, $B) // Same as above
func F[$B, $A interface{}]($A, $B) // Same as above

func F[$A C]($A) // $A is constrained by C

func F[$A C]($A, $B) // $A, $B are constrained by C, interface{}, respectively
func F[$B C]($A, $B) // $A, $B are constrained by interface{}, C, respectively

func F[$A C, $B C]($A, $B) // $A, $B are constrained by C
func F[$A, $B C]($A, $B) // Same as above

func F[$A C1, $B C2]($A, $B) // $A, $B are constrained by C1, C2, respectively

func F[$A, $B C1, $C, $D C2]($A, $B, $C, $D) // $A, $B and $C, $D are constrained by C1 and C2, respectively
func F[$D, $C C2, $B, $A C1]($A, $B, $C, $D) // Same as above

func F[$A, $B, $C C]($A, $B, $C) // $A, $B, $C are constrained by C

// Method declarations

func (T) M($A) {}
func (T) M[$A interface{}]($A) {}
// ...etc. See function declarations above.

// Method signatures

interface { M($A) }
interface { M[$A interface{}]($A) }
// ...etc. See function declarations above.

// Function literals

func($A) {}
func[$A interface{}]($A) {}
// ...etc. See function declarations above.

// Nested type-to-value abstractions

func F(a $A) $A {
    // $A below is bound by the function literal
    return func(a $A) $A { return a }(a)
} // Valid

func F(a $A, b $B) ($A, $B) {
    // $A below is bound by the function literal
    // $B below is bound by F
    return func(a $A) ($A, $B) { return a, b }(a)
} // Valid

// Parametricity

func F(a, b $A) $A {
    choose := func(b bool) $A {
        var result $A
        if b {
            result = a
        } else {
            result = b
        }
        return result
    }
    return choose(a == b)
} // Valid

// Shorthand for combining multiple constraints into one

func F[$A C]($A)
func F[$A interface { C }]($A) // Same as above

func F[$A interface { C1; C2 }]($A)
func F[$A (C1, C2)]($A) // Same as above

func F[$A interface { C1; C2; C3 }]($A)
func F[$A (C1, C2, C3)]($A) // Same as above
```

*Add type-to-value abstraction types to types.*

A type-to-value abstraction's type is comprised of an unordered set of type parameters and their constraints, and the type of the abstracted function value. Two such types are identical if their type parameters, constraints, and function type are identical. One such type is assignable to another such type if they are identical, or if they can be made identical after [renaming](https://en.wikipedia.org/wiki/Lambda_calculus#Alpha_equivalence) the type parameters in one of them and the corresponding constraints are assignable.

# TODO
Type-to-value abstraction types are represented syntactically

```go
type Box[$T] struct {
    X $T
}

func NewBox() Box[$T] {
    return Box{}
}

var b Box[$T] = NewBox()
b.X = 123
$app(b, int).X = 123
```

Examples:

```go
// Assignable because of identical types

var _ func($A) = func($A) {}
var _ func($A, $B) = func($A, $B) {}

var _ func[$A C]($A) = func[$A C]($A) {}
var _ func[$A C1, $B C2]($A, $B) = func[$A C1, $B C2]($A, $B) {}

// Assignable because of equivalent type variables

var _ func($A) = func($B) {}
var _ func($A, $B) = func($C, $D) {}

var _ func[$A C]($A) = func[$B C]($B) {}
var _ func[$A C1, $B C2]($A, $B) = func[$C C1, $D C2]($C, $D) {}

// Assignable because of equivalent constraints

var _ func[$A interface{}]($A) = func[$A interface{}]($A) {}
var _ func[$A interface{ M() }]($A) = func[$A interface{}]($A) {}
var _ func[$A interface{ M1(); M2() }]($A) = func[$A interface{ M1() }]($A) {}
```

*Add type-to-type abstractions to type declarations.*

A type-to-type abstraction abstracts the underlying type of the type specified by a type declaration. It is implied by specifying a list of one or more type parameters and their constraints. Constraints default to an empty interface if unspecified. The identifier for such a declared type is called a **type constructor** when used to construct types, and a **value constructor** when used to construct values. The body cannot be a lone type variable or a lone recursive reference.

# TODO: should func types bind tyvars in param types like in type-to-value abs?
# TODO: what happens for package-scope var of generic type? what happens when generic field assigned to with diff types?
var v List[$T] = NewList() // require concrete types in var types instead of tyvars?

Examples:

```go
// [...] is a non-empty, ordered set of type parameters and constraints

type T[$A] int // Valid
type T[$A] $A // Invalid

type T func($A) $A // T is not generic, but its underlying type is
type T[$A] func($A) $A // T is generic, but its underlying type is not
type T[$A] func($B) $B // T and its underlying type are generic

type T[$A] func($A) $A // $A is constrained by interface{}
type T[$A interface{}] func($A) $A // Same as above
type T[$A C] func($A) $A // $A is constrained by C

type T[$A C, $B C] func($A, $B) ($A, $B) // $A and $B are constrained by C
type T[$A, $B C] func($A, $B) ($A, $B) // Same as above

// $A and $B are constrained by C1
// $C is constrained by C2
// $D and $E are constrained by C3
// $E and $F are constrained by interface{}
type T[$A, $B C1, $C C2, $D, $E C3, $E, $F] func($A) ($A)
```

### Type applications

*Add **type applications** to types and expressions.*

Type applications **apply** type abstractions to type arguments, in which they substitute type arguments for bound type parameters within type abstraction bodies. There are two kinds of type application: values that substitute types (type-to-value) and types that substitute types (type-to-type). Each kind of type application is used with its corresponding kind of type abstraction.

The number of type parameters and type arguments must match. Type arguments must satisfy constraints.

Type constructors used as a type, or within a type, that aren't applied to type arguments are **raw** types, where they are implicitly applied to their type parameter constraints, and are thus assignable and convertible to those same explicit type applications.

# TODO: returning result of data constructor that didn't specify tyvar field. returned generic value is still wrapped by tyabs. later assignment to tyvar field is a tyapp. assignment of each field separately is equivalent to calling ctor with all fields specified; assignment is mutation of state by composition of state-mutation functions like assignment. e.g. `type Box[$T] struct { X $T; Y int }; func NewBox() Box[$T] { return Box{} }; var box Box[$other] = NewBox(); box.Y = 34 /* box type is still Box[$other] */; box.X = 3.4 /* box type is now Box[float64] */`. or rather the compiler looked ahead, saw the assignment of box.X to 3.4, equated $other with float64, and box was always declared as Box[float64].

# TODO: types AND values are implicit. talk about inference for type of ty-to-val values.

# TODO: The order of type parameters determines the order of type arguments. 

# TODO: higher-order types? func($A[$B])?

# TODO
explain how syntax implicitly specifies order of tyabs typarams, tyapp tyargs

# TODO:
```go
// Parameterization

func FirstInt(List[int]) int
func FirstGeneric(List[$A]) $A

func (List[int]) FirstInt() int
func (List[$A]) FirstGeneric() $A

func Find(List[int], int) (int, bool)
func Find(List[$element], $element) ($element, bool)

func Find(List[int], int) (int, bool)
func Find(List[$element], $element) ($element, bool)
```

#### Type-to-value applications

For function and method calls, type applications are **implicit**. Type arguments are inferred from function arguments, and formed in the same deterministic order as the implicit type parameters for the type abstraction, whether the type abstraction is the function value's type or its type's underlying type. Addressable values are referenced (the address taken of) automatically where it aids satisfying constraints.

---

@@@@@@@@@@@@@@@@@@@@@@@@@@@@ TODO:

assignability and typeconversion of tyabs where some tyvars are filled in

---

Where type-to-value type abstractions are assigned or converted to a function type, type arguments are inferred from the function type, and the abstractions are wrapped in a type-to-value type application 

**inserted**

 or another type abstraction that could be made identical with type parameter renaming or if one or more type parameters are substituted with type arguments, those values are wrapped in a type-to-value type application that applies them to those type arguments. If there are any remaining type parameters, the type application is wrapped in a type-to-value type abstraction they are abstracted in 

applies to method matching in interfaces

```
var f func(int) int = func(a $A) $A { return a }
```

Examples:

```go
func identity(a $A) $A { return a }
var _ = identity(123) + 456

func constant(a $A, b $B) $A { return a }
var _ = constant(123, "foo") + 456

func (l *List[$A]) Map(f func($A) $B) *List[$B] { return nil }
var _ *List[string] = myIntList.Map(func(n int) string { return strconv.Itoa(n) })
```

#### Type-to-type applications

Type-to-type type applications apply type-to-type abstractions to type arguments.

Type-to-type type applications are **explicit**. 

For types, type applications are **explicit**. The type abstraction must be a type identifier.

Type applications for types can be type converted to and from their underlying type with the equivalent type argument substitutions made within the underlying type.

TODO: tyapps substitute tyargs for typarams inside tyabs bodies, the underlying types. tyargs are part of ty identity. full types resulting from tyapp of tycon to args is represented by the tyapp. tyapps substitute tyargs for tyvars within underlying type, and retain tyargs in type for identity.

TODO: Reflection can reveal the type arguments for generic types.

Examples:

```
A[B]
A[$B]
$A[B]
$A[$B]
A[B, C]
A[B, C, D]
A[B[C], $D[$E]]
```

# TODO:
```go
// Abstraction with type variables

func FirstInt(List[int]) int
var FirstIntFunc func(List[int]) int = FirstInt

func FirstGeneric(List[$element]) $element
var FirstGenericFunc func(List[$element]) $element = FirstGeneric

func (List[int]) First() int
func (List[$element]) FirstGeneric() $element
var FirstGenericMethodVar = List

func Find(List[int], int) (int, bool)
func Find(List[$element], $element) ($element, bool)

func Find(List[int], int) (int, bool)
func Find(List[$element], $element) ($element, bool)
```

### Raw types

# TODO

### Parameterized interface method receivers

*Change type operations to methods.*

Add an optional receiver parameter to interface methods. They appear before method names in a separate parameter section, like in method declarations. They are like normal method parameters, except the type must either be a type variable or a type variable applied to type arguments. They are subject to the same explicit type parameters as other method parameters.

@@@@@@@@@@@@@@@@@@@@@@@@@@@@ TODO:

method values are tyapp of receiver type to tyabs wrapping the func value, wrapped in tyabs for remaining ty params

generic interface methods without receivers still match methods of generic types like
func (List[$A]) Append($A)

interface method receivers only permitted for constraint interfaces, because those are teh only place where the receiver tyvar is accessible if needed

method values for generic receivers keep knowledge of tyvars in other params and they must match

problem with `var i Iterable = ...` is that the Iterable isn't a parameter. the reason why Number works is because there are multiple params with same tyvar, so they can work together. you're not supposed to know what the concrete tyvar type arg is. so...apparently you can't use interfaces that have receivers outside of func params...damn. but generic methods that don't have params should work just fine. but this might contradict an above concern i have with the go team's proposal where type lists can only be used in constraints, not normal interfaces.

invalid or at least stupid to list receiver tyabs ident in tyapp in constraints. probably invalid, since you could list constraints not satisfied by the interface.

Examples:

```go
interface { M() }
interface { $A M() } // Same as above
interface { ($A) M() } // Same as above
interface { (a $A) M() } // Same as above

interface { $A M($A) }
interface { $A M($A) $A }

interface { (x $N) Max[$N Number] (y $N) (z $N) }

interface { $container[$element] Contains($element) bool }
```

### Unify defined and built-in types

Create a new (real) standard library package named `builtin`. Every [file block](https://golang.org/ref/spec#Blocks) implicitly has the declaration `import . "builtin"`.

Move predeclared types to `builtin` as exported declarations (with capitalized names) (e.g. `Bool`, `Int`, `Rune`, `String`). Change identifiers in the [universe block](https://golang.org/ref/spec#Blocks) for predeclared types to be [aliases](https://golang.org/ref/spec#Type_declarations) for their corresponding exported declaration in `builtin`.

Add generic type declarations to `builtin` for primitive generic types, such as:

- `type Chan[$A]`
- `type Map[$K Equatable, $V]`
- `type ReceiveChan[$A]`
- `type SendChan[$A]`
- `type Slice[$A]`

Make primitive generic types syntactic sugar for their `builtin` counterparts, such as:

- `chan A` => `Chan[A]`
- `map[A]B` => `Map[A, B]`
- `<-chan A` => `ReceiveChan[A]`
- `chan<- A` => `SendChan[A]`
- `[]A` => `Slice[A]`

Array types have a special, predeclared value-to-type value abstraction in the universe block, since array types contain constant values (the length). This makes array type constructors, and applying array types, more readable.

Examples:

```go
array(10) // Type constructor for an array of size 10
array(N) // Type constructor for an array of size N
array(N)[Int] // Array of size N of Int
```

Make array types syntactic sugar for `array` types, such as:

- `[10]Int` => `array(10)[Int]`
- `[10]$A` => `array(10)[$A]`
- `[n]Int` => `array(n)[Int]`
- `[n]$A` => `array(n)[$A]`

Add whatever constants, variables, functions, and methods to `builtin` that may be useful, with generics or otherwise, such as:

- `func (Int) Sum(Int) Int`
- `func (String) Equals(String) Bool`
- `func (Rune) Less(Rune) Bool`
- `func (Chan[$A]) Len() Int`
- `func (Chan[$A]) ReceiveChan() ReceiveChan[$A]`

### Alias type constructors -- TODO: Type aliases?

Add type constructors to type aliases to enable aliasing generic types and further constraining type parameters. Unapplied alias type constructors are raw types, like for non-alias type constructors.

TODO: are these tyabs?

Examples:

```go
type Dictionary = Map // Dictionary == Map, Dictionary[$A, $B] == Map[$A, $B]

type Callback[$A] = func() $A
type Coordinate[$A Number] = [2]$A
type Counter[$A Equatable] = Map[$A, Int]
type List[$A Orderable] = []$A
type Reference[$A] = *$A
type Set[$A Equatable] = Map[$A, struct{}]
```

### Examples

```go
package builtin // import "builtin"

// ...

// Iteration

type Iterable[$A] interface {
    Iterator() Iterator[$A]
}

type Iterator[$A] interface {
    HasNext() Bool
    Next() $A
}

// Slice[$A] implements Iterable[$A]

func (s Slice[$A]) Iterator() Iterator[$A] {
    return &sliceIterator[$A]{s: s}
}

type sliceIterator[$A] struct {
    i Int
    s Slice[$A]
}

func (i sliceIterator[$A]) HasNext() Bool {
    return i.i < len(i.s)
}

func (i *sliceIterator[$A]) Next() $A {
    a := i.s[i.i]
    i.i++
    return a
}

// Numbers

type Equatable interface {
    $A Equals($A) Bool
    $A NotEquals($A) Bool
}

type Orderable interface {
    Equatable
    $A Greater($A) Bool
    $A GreaterEquals[$A Equatable] ($A) Bool
    $A Less($A) Bool
    $A LessEquals[$A Equatable] ($A) Bool
}

type Number interface {
    Orderable
    $A Difference($A) $A
    $A Product($A) $A
    $A Quotient($A) $A
    $A Sum($A) $A
}

// Int implements Number
// Implementations are internal to the compiler or runtime

func (Int) Equals(Int) Bool
func (Int) NotEquals(Int) Bool

func (Int) Greater(Int) Bool
func (Int) GreaterEquals(Int) Bool
func (Int) Less(Int) Bool
func (Int) LessEquals(Int) Bool

func (Int) Difference(Int) Int
func (Int) Product(Int) Int
func (Int) Quotient(Int) Int
func (Int) Sum(Int) Int

package list // import "collection/list"

// ...

// List[$A] implements Iterable[$A]

func (l *List[$A]) Iterator() Iterator[$A] {
    return &listIterator[$A]{e: l.Front()}
}

type listIterator[$A] struct {
    e *Element[$A]
}

func (i listIterator[$A]) HasNext() Bool {
    return i.e != nil
}

func (i *listIterator[$A]) Next() $A {
    a := i.e.Value
    i.e = i.e.Next()
    return a
}

package mypackage

// Summation for any Number iteration, including Slices and Lists

func Sum[$A Number](i Iterable[$A]) $A {
    var sum $A
    for i.HasNext() {
        sum = sum.Sum(i.Next())
    }
    return sum
}

var sliceSum Int = Sum(myIntSlice)
var listSum Int = Sum(myIntList)
```

### Operations become inherited methods

TODO

---

TODO: maybe tyapps for constraints should be allowed?

```
type Node[$E] interface {
    Edges() []$E
}

type Edge[$N] interface {
    Nodes() (from, to $N)
}

type Graph[$N, $E] = graph[$N[$E], $E[$N]]

type graph[$N, $E] struct { ... }

func (g *graph[$N, $E]) ShortestPath[$N, $E](from, to $N) []$E { ... }

//////////////

type Node[$E Edge[Node]] interface {
    Edges() []$E
}

type Edge[$N Node[Edge]] interface {
    Nodes() (from, to $N)
}

type Graph[$N Node, $E Edge] struct { ... }

func (g *Graph[$N, $E]) ShortestPath[$N Node, $E Edge](from, to $N) []$E { ... }

type MyNode struct { ... }
func (*MyNode) Edges() []*MyEdge { ... }

type MyEdge struct { ... }
func (*MyEdge) Nodes() (from, to *MyNode) { ... }

var myGraph *Graph[*MyNode, *MyEdge] = &Graph[*MyNode, *MyEdge]{ ... }

//////////

type Node[$E Edge] interface {
    $N Edges() []$E[$N]
}

type Edge[$N Node] interface {
    $E Nodes() (from, to $N[$E])
}
```

---

generic types can't have constructor funcs, must use init methods

int and Int are interchangeable, i just use Int to be consistent here

doc can elide `builtin.` qualifiers from syntax

nil tyvar values for results become zero values for type arg

change Iterable example to use Iterable[$A] and Get($A)

methods aren't special, can be generic

extended interfaces and generics removes need for type lists

I left out a lot of reasons, feel free to ask

http://homepages.inf.ed.ac.uk/wadler/papers/gj-oopsla/gj-oopsla-letter.pdf

not as powerful as haskell:
    interfaces work on values, not types, so cannot have custom constructors or values (like having '1' be a Number, not an int)

other unifications/simplifications:
    tuples
    change func to be in terms of tuples or now others
    sum types, and better error handling
    remove underlying types
    remove addressability, every value can be made addressable on demand
    make arith ops compatible with methods, take away untyped results
    change built-in funcs to methods
    parameterized aliases
    make maps not reference types
    funcs always return results, just as struct{} if needed
    remove capacity from slices
    operators as syntactic sugar for generic methods
    function currying
    tail-call optimization
    funcs have one param and one result, func types have two params, add tuples to be compatible
    make some operations like arith into syntactic sugar for equivalent methods
    make pointer types be default:
        everything is passed by value, and by reference is opt-in with pointers
        but most user-defined types are structs, and without knowing unexported fields, it's safest to always pass by ref
        so pointers almost always used with passed types
        but sometimes pass by value is good
        so it seems we need a way to specify default pass policy for a type
        but also a way to override whatever the default pass policy for a type is in a particular case
        e.g. sometimes we want to pass an int pointer, or a non-pointer struct
        it seems like a way to enforce immutability is to simply provide a way to prohibit using pass by ref
        and then have all mutations (like struct field assignment, as if it's a func/method) require pass by ref

type abs for types type param order matters because explicit type args required in type apps, and those are required because zero values for types mean that not all subvalues need to be provided, so those types can't be inferred.

builtin doc doesn't show internal type details, just `type Int`, `type String`, etc. it does list methods and associated funcs, vars, and consts like normal.

i don't care if:
    it's called prelude or core or base or whatever instead of builtin
    it's @ or # or ! or whatever instead of $
    it's parentheses or curly brackets or parentheses inside square brackets or whatever instead of square brackets
    it's called type instantiation instead of type application
    it's called something else instead of type abstraction, type con, etc.

inline func/method calls where possible

smart planning for method names can allow existing stdlib types to be made compatible with constraints, like math/big.Int with Number. i wouldn't be surprised if breaking changes are required to make everything consistent and work together.

tyvars are syntactically diff from just interfaces, so other impl strategies are possible without being inconsistent. worse runtime perf when using generics is consistent with current practice of using interfaces and boxing.

how tyabs are implemented (are they really values?) is up to the implementors

no explict tyapps for funcs

reasoning about it as type erasure is possible, and compiling it that way is possible

using [unification](https://en.wikipedia.org/wiki/Unification_(computer_science))

for unifying declared and builtin types:
    There's a lot more that could be unified and simplified, but that's probably beyond the scope of just generics.

pointer types removed 

haskell practice is to name tyvars from a to z, and use first letter of type classes in contexts. seems like good practice.

generic type aliases are required to be complete. go ahead and try to make an argument to the haskell community that the corresponding haskell feature, type synonums, isn't necessary and should be removed, and see how far you get.

tyabs types and values, and tyapp evaluation, might not exist at runtime, might only exist at compile time

it should be possible to impl this without breaking compat

## Upsides

I think this approach solves these problems:

- A new feature like type sets isn't required; rather, an existing feature is enriched
- Parametric methods
- Backward compatibility ?????? raw types ??????
- TODO

I'm including this because it might motivate the reader to read my proposal below. If you don't care, then skip to the next section for the start of my proposal.

Parts of the Go Team's proposal are inconsistent with existing Go or with each other:

- Currently, to reuse code with varying types, we have two options: (1) use interfaces instead of concrete types, boxing, and unsafe type assertions, or (2) copy code using concrete types, no boxing, and no unsafe type assertions. The proposal adds type safety to the former practice, but [doesn't use boxing](https://go.googlesource.com/proposal/+/refs/heads/master/design/43651-type-parameters.md#values-of-type-parameters-are-not-boxed), so we would now have two common ways to reuse code with varying types, each with different space and time trade-offs.
- It says the compiler could use either slow compile times or slow execution times \[[1](https://go.googlesource.com/proposal/+/refs/heads/master/design/43651-type-parameters.md#implementation)\] \[[2](https://go.googlesource.com/proposal/+/refs/heads/master/design/43651-type-parameters.md#efficiency)\], but it also says that [values of type parameters won't be boxed](https://go.googlesource.com/proposal/+/refs/heads/master/design/43651-type-parameters.md#values-of-type-parameters-are-not-boxed), which seems to mean compile times will always be slow.
- Currently, interfaces only describe behavior, which any type might have, but type lists explicitly state which concrete types (or concrete underlying types) are allowed. Also, I'm not aware of another context where a type identifier is allowed to match an underlying type instead of the "overlying" type.
- You can't limit which operations a type list makes available (e.g. only subtraction). This runs counter to using narrow interfaces that only include the needed behavior (e.g. `io.Reader` versus `io.ReadCloser`). Rather than having a precise way to specify operations, we have to list exhaustive examples and hope that the compiler can guess what we mean.
- Type lists enable interfaces to provide access to struct fields. If Manager and Employee struct types have name fields, and we want to write a function that can get the name from either type, we already have a way to do that: add a method to both types, and make an interface that describes that method. Now there are two ways to do that, and it's unclear which way is best in which situations. If type lists are included at all, they should be limited to operations that are function-like, like math operators.
- Generic slice elements [aren't addressable](https://go.googlesource.com/proposal/+/refs/heads/master/design/43651-type-parameters.md#pointer-method-example), even though [non-generic slice elements are](https://play.golang.org/p/OkuFh_IM8hg).

Parts of it are complicated:

- Constraints are interfaces, but [type lists can only be used in constraint interfaces](https://go.googlesource.com/proposal/+/refs/heads/master/design/43651-type-parameters.md#type-lists-in-interface-types). Constraint interfaces are basically a new language construct, like contracts were, except they share the interface name, which is worse because it's confusing. Now certain interfaces are invalid in certain contexts.
- Constraint interfaces must always be specified, even though `interface{}` is a reasonable default. This will clutter programs. `any`, being shorter than `interface{}`, seems like a crutch for minimizing the clutter. It's unclear why `any` isn't permitted outside of constraints.
- [Generic recursive types must use the same type arguments in the same order](https://go.googlesource.com/proposal/+/refs/heads/master/design/43651-type-parameters.md#generic-types), and it's unclear why. It says, "This restriction prevents infinite recursion of type instantiation," but the example wraps with a pointer, so the value size is known and finite.

    It works just fine in Haskell:

    ```hs
    data MyType tyvarA tyvarB = MyDataConstructor (MyType tyvarB tyvarA)
    ```

    and in Java:

    ```java
    class MyType<tyvarA, tyvarB> {
        private MyType<tyvarB, tyvarA> myDataField;
    }
    ```

- Methods can't be generic, even though [method expressions](https://golang.org/ref/spec#Method_expressions) and [method values](https://golang.org/ref/spec#Method_values) are [normal functions](https://play.golang.org/p/-QGzJLDDddB):

    ```go
    type T struct{}
    func (T) M(T) {}
    var F func(T, T) = T.M
    var M func(T) = T{}.M
    ```
- It [adds a special comparable constraint](https://go.googlesource.com/proposal/+/refs/heads/master/design/43651-type-parameters.md#comparable-types-in-constraints) because type lists aren't expressive enough to specify comparison operators for composite types. It's unclear, but it seems like this means that the type list `type int` doesn't include `int` comparison operators.

Parts of it are confusing:

- If an interface only contains a type list, then its method set is empty, but it's not an "empty interface". How will that be indicated in reflection? What new term should we use to distinguish such interfaces from empty interfaces?
- It's unclear what the difference is between `func[X interface{}](X)` and `func(interface{})`.
- It's possible to [construct types that no values can match](https://go.googlesource.com/proposal/+/refs/heads/master/design/43651-type-parameters.md#comparable-types-in-constraints).
- [The last type arguments in a type instantiation can sometimes be omitted](https://go.googlesource.com/proposal/+/refs/heads/master/design/43651-type-parameters.md#type-inference), but it's not obvious from the code being typed when that's the case. This makes me wonder how easy it will be to break compatibility by accidentally adding an extra type parameter that can't be inferred.
- Interfaces with a type list and a method can never be satisfied by the exact types listed.
- It doesn't require [parametricity](https://en.wikipedia.org/wiki/Parametricity), so [some odd problems arise](https://go.googlesource.com/proposal/+/refs/heads/master/design/43651-type-parameters.md#pointer-method-example), and the means of overcoming them are difficult to read and understand. Generic functions are [permitted to create generic values](https://go.googlesource.com/proposal/+/refs/heads/master/design/43651-type-parameters.md#pointer-method-example), which causes all kinds of complications with addressability, method sets, untyped constants, type conversions, and zero values. Having type variables not associated with function parameters causes problems with type inference and requires explict type instantiations.
- [Defined types are expected to be interchangeable with their underlying type](https://go.googlesource.com/proposal/+/refs/heads/master/design/43651-type-parameters.md#element-constraint-example), even though they are different. It seems correct to require defined types to be type converted to their underlying type for generic code that only works with their underlying type. The only reason why a defined type can be used as its underlying type in a function call is due to [assignability](https://golang.org/ref/spec#Assignability).
- >This restriction may be lifted in future language versions. An interface type with a type list may be useful as a form of sum type, albeit one that can have the value nil.

    Is nil a valid value for a type variable with this current proposal? I thought [values of type parameters are not boxed](https://go.googlesource.com/proposal/+/refs/heads/master/design/43651-type-parameters.md#values-of-type-parameters-are-not-boxed)? If it's invalid, then this seems to be saying that nil *would* be valid outside of constraint contexts.
- >If a generic function or type is used without specifying all the type arguments, it is an error if any of the unspecified type arguments cannot be inferred.

    It's unclear when type arguments can't be inferred. Is it when they're only used in the function body or result types?

Parts of it are incomplete:

- [It doesn't enable generic functions to be used as values](https://go.googlesource.com/proposal/+/refs/heads/master/design/43651-type-parameters.md#instantiating-a-function).
- There's no way to describe common generic methods with defined types, like comparisons or ordering. [You can only do it with an interface literal as a constraint](https://go.googlesource.com/proposal/+/refs/heads/master/design/43651-type-parameters.md#using-types-that-refer-to-themselves-in-constraints).
- What happens when a method name and a field name in a type list conflict in a constraint interface?

    ```
    var x interface {
	    type struct { A func() }
        A()
    } = y
    x.A() // Which A is used?
    ```
- It's unclear whether [channels are permitted by `comparable`](https://go.googlesource.com/proposal/+/refs/heads/master/design/43651-type-parameters.md#comparable-types-in-constraints).

Other comments:

- >A type argument satisfies a type constraint with a type list if the type argument or its underlying type is present in that type list. If in some future language version we permit interface types with type lists outside of type constraints, this rule will make them usable both as type constraints and as sum types.

    I thought type assertions for interface values made sum types redundant? This was the reason the Go Team gave us, or at least me, for not adding sum types. If we're going to consider adding sum types, then let's have that broader discussion *now*, before we inextricably link them to interfaces and ad-hoc polymorphism, which are unrelated concepts.

Below is a proposal for a design that aims to avoid or fix these problems. I tried to make this as accessible as possible for the Go Community, not just the Go Team. That being said, I tried to keep it at a high level to be concise.

## Downsides

TODO

## Remarks

In my opinion, [Haskell](https://en.wikipedia.org/wiki/Haskell_(programming_language)) has the gold standard of type systems, so let's add its [ad-hoc polymorphism](https://en.wikipedia.org/wiki/Ad_hoc_polymorphism), [parametric polymorphism](https://en.wikipedia.org/wiki/Parametric_polymorphism), and [type operator](https://en.wikipedia.org/wiki/Type_constructor) features to Go. Java did something similar with Haskell's parametric polymorphism and type operator features, which Java called [generics](https://en.wikipedia.org/wiki/Generics_in_Java). Incidentally, [Haskell's ad-hoc polymorphism feature](https://en.wikipedia.org/wiki/Type_class) and Java's generics features were worked on by [Philip Wadler](https://en.wikipedia.org/wiki/Philip_Wadler), with whom the Go Team collaborated \[[1](https://www.youtube.com/watch?v=Dq0WFigax_c)\] \[[2](https://arxiv.org/abs/2005.11710)\] when thinking about generics for Go, and presumably when putting together their generics proposal.

Since Go is in a position similar to the one Java was in before it added generics, I followed Java's general approach, adapted to Go, and extended to be more powerful than Java, although not as powerful as Haskell.

Since Go functions are a kind of value abstraction (values that abstract values, called **value-to-value abstractions**), adding both kinds of type abstraction to Go makes it only [one dimension away from full type and value generalization](https://en.wikipedia.org/wiki/Lambda_cube), lacking only the other kind of value abstraction, [types that abstract values](https://en.wikipedia.org/wiki/Dependent_type), called **value-to-type abstractions**, array sizes notwithstanding.

TODO: in remarks: not possible to declare a tyabs for func/method that contains no tyvars in body
TODO: in remarks: discuss copy/paste safety of having tyvars in param types auto create tyabs
TODO: in remarks: if generics for func sigs doesn't make sense, then remove; just included to be complete

## Breaking compatibility wishlist

- Remove the universe block
    - Remove all predeclared identifiers
        - Change `nil` to a keyword that cannot be an identifier
        - Add sum types to express `true` and `false`
        - Remove `print` and `println`
        - Add function counterparts to the standard library for all others
        - Convert all others to syntactic sugar for those counterparts
- Make all struct type declarations be function declarations
    - Match struct field and func param decls, both are the same, structs can have varargs at end
    - Convert builtin constructors like make to these functions, use optional "fields" for optional make params

Make interfaces only usable as constraints, then you can drop the always-boxing semantics; although it breaks Println([]interface{})

Remove complex types

## Related

### Sum types

```go
type Tree[$T] switch {
    Empty
    Node($T, Tree[$T], Tree[$T])
}

type Direction switch {
    North
    East
    South
    West
}

var _ Direction = Direction.North

type Direction switch {
    North()
    East()
    South()
    West()
}

type Person switch {
    Manager(Name string)
}

switch v := p.(type) {
case Manager, Person.Manager:
    fmt.Println("Manager")
case Manager(name):
    fmt.Println(name)
case Foo(x, y int) {
    
}
}

func Sort(as []$A, Asc bool) {

}
```

# allow conversion between structs with same fields except diff names
make func params, results be implicit struct decls

#numeric conversion
t(u) does literal conversion of bits, whether signed or unsigned ???