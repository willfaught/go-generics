# Enrich the type system

## Add bounded parametric polymorphism and higher-order types

### New types

New syntax:

- Type variables: `V`
- Type-to-type abstractions: `func(type V I) T`
    - `V` is a type variable
    - `I` is any interface or identifier whose underlying type is an interface
    - `T` is any type
    - `V` can be repeated in a comma list
    - `I` can be omitted and thereby default to `interface{}`
    - `V I` can be repeated in a comma list (trailing comma allowed)
- Type-to-type applications: `T1(T2)`
    - `T1` and `T2` are any type
    - `T2` can be repeated in a comma list (trailing comma allowed)

```
func Print(type T fmt.Stringer) (xs ...T) (int, error) // ...
func (type T fmt.Stringer) (xs ...T) (int, error) // ...

type List(type T fmt.Stringer) []T
```

New semantics:

The number of type variables in the parameter and argument lists of a type-to-type application must match. A type-to-type application substitutes the type arguments for free occurrences of their corresponding type parameters in the type-to-type abstraction body. All type-to-type abstractions must be applied. It is a type error for a type-to-type abstraction to remain in a variable type at the end of type evaluation.

Examples:

```
func[[T]] func(T) T
List[[T]]

var xs List{{int}}
var xs List[[int]]
xs.Append(1)
xs.Append(2)


type List func(type T) []T
func(type T) Append(l *List(type T), t T) // ...
func(type T)(l *List(type T), t T) // ...
```

New syntactic sugar:

```
type List func(type T) []T
type List func(T) []T


type List(type T) struct {
    Sort func(type T ordered) func([]T) []T
}

type Comparable interface {
    T Equal(type T) (T) bool
}

func (type T) (l *List(T)) Append(t T) // ...

func InsertionSort(type T orderable) (ts []T) { // ...
func SelectionSort(type T orderable) (ts []T) { // ...

func[T orderable] InsertionSort (ts []T) { // ...
func[T orderable] SelectionSort (ts []T) { // ...

func[A orderable, B comparable, C interface{orderable; comparable}] SelectionSort (ts []T) {
    
}

func SelectionSort [A orderable, B comparable, C interface{orderable; comparable}] (ts []T) {
    
}

func TestSorts(t *testing.T) {
    for i, f := range []func(type T orderable) ([]T) {
        InsertionSort,
        SelectionSort,
    } {
        a, e := []int{3, 2, 1}, []int{1, 2, 3}
        f(int)(a)
        if !reflect.DeepEqual(a, e) {
            t.Error(i, a, e)
        }
    }
}

func TestSorts(t *testing.T) {
    for i, f := range []func[T orderable] func([]T) {
        InsertionSort,
        SelectionSort,
    } {
        a, e := []int{3, 2, 1}, []int{1, 2, 3}
        f[int](a)
        if !reflect.DeepEqual(a, e) {
            t.Error(i, a, e)
        }
    }
}

type Foo struct {
    r io.Reader
    w io.Writer
    count int
    buf []interface{}
}

type Foo(type A io.Reader, B io.Writer, C interface{}) struct {
    r A
    w B
    count int
    buf []C
}

func Foo(r io.Reader, w io.Writer, buf []interface{})

func Foo(type A io.Reader, B io.Writer, C interface{}) (r A, w B, count int, buf []C)

func Concat(type T) (a, b List(T))

func Print(type T fmt.Stringer) (s []T) {
    for _, v := range s {
		fmt.Println(v)
	}
}

func Print(s []fmt.Stringer) {
    for _, v := range s {
		fmt.Println(v)
	}
}

Print(int)([]fmt.Stringer{a, b, c})

func Const(a, b interface{}) interface{} {
    return a
}

func Const(type A, B) (a A, b B) A {
    return a
}

// reconstructed from all interfaces for args/results
func Const(type A, B) (a A, b B) interface{} {
    return a
}
```

legacy types equivalent to bounded polymorphic types in params only, can't tell in general what interface results correspond to what tyvars, and it doesn't make sense to break parametricity

tyvars in funcs:
    when not applying func to type first, tyvars degrade to their constraint types, same as compiled code for func
    when applied, tyvars replaced with args for type checking, and conversion/constraints on args and results

```
func Const(type A, B) (a A, b B) A {
    return a
}
```

Const() has type func(interface{}, interface{}) interface{}
Const(int, string) has type func(int, string) int, but compiles to func(interface{}, interface{}) interface{} with automatic conversions and assertions at call sites

funcs with tyvars can degrade to param/result types that are just interfaces
    `func Const(type A, B) (a A, b B) A` degrades to `func(interface{}, interface{}) interface{}`

func Const(type A, B) (a A, b B) A {
    return a
}

var f func(type A, B) (A, B) A = Const
var g func(int, string) int = Const(int, string)

### New expressions

New syntax:

Type variables: V
Type-to-value abstractions: func(type V I) E
    V is a type variable
    I is any interface or identifier whose underlying type is an interface
    E is any expression
    V can be repeated in a comma list
    I can be omitted and thereby default to interface{}
    V I can be repeated in a comma list
Type-to-value applications: E(type T)
    T1 and T2 are any type
    T2 can be repeated in a comma list

New semantics:



Examples:

(func(type A, B) func(f func(A) B, as []A) []B)(type int, string)

Syntactic sugar:

(func(f func($A) $B, as []$A) []$B)(type int, string)

### New declarations

New syntax:

Type variables: V
Type-to-declaration abstractions: func(type V I) D
    V is a type variable
    I is any interface or identifier whose underlying type is an interface
    D is any function or type declaration
    V can be repeated in a comma list
    I can be omitted and thereby default to interface{}
    V I can be repeated in a comma list
Type-to-declaration applications: D(type T)
    D is any function, type, or variable declaration
    T is any type

New semantics:



Examples:

func(type T) Identity(x T) T { return x }
Identity(type int)(7)

type Set func(type T) map[T]struct{}
Set(type int){}[7]

Syntactic sugar:

func Identity(x $T) $T { return x }
Identity($int)(7)

type Set map[$T]struct{}

---

```
type Functor(type F) interface {
    F(A) Map(f func(A, B)) F(B)
}

type Set(type T) []T

func (type A, B) (s Set(A)) Map(f func(A, B)) Set(B) {
    bs := make(Set(B), len(s))
    for i := range s {
        bs[i] = f(s[i])
    }
    return bs
}

type Monad(type M) interface {
    M(A) Bind(func(A) M(B)) M(B)
    M(A) Sequence(M(B)) M(B)
}
```



## Adapt the predeclared types to the new features

```
func Min(x, y T) T
func(type T) Min(x, y T) T
func(type T) Min(x, y T) T
func(type T) (x, y T) T

type List(T) []T
type List func[T] []T
List[int]

func[T] Min(x, y T) T
Min[int](2, 3)

func[T](x, y T) T


func Min[T orderable](x, y T) T {
    if x < y { // if x.Less(y)
        return x
    }
    return y
}

type List func[T] []T

List[int] => (func[T] []T)[int] => []int

type List[T] []T

type[T] List []T

type[T] (
    List []T
)

type (
    List[T] []T
)

func (l *List[T]) Append[T](t T) P
    *l = append(*l, t)
}

in type or func decls, put ty params in square brackets next to identifiers

func Min[T orderable](x, y T) T
func[T orderable](x, y T) T

specific tyvars don't matter when it comes to assignability and interface impl, alpha conversion is fine

type Foo[T] struct {
    history List[T]
    buf []T Slice[T] Array[5, T]
    next Heap[T]
}

tyvars in func results must also be in params

func Helper(f func[A, B](A) B) (err error) {

}

since all tyvars must be able to implement interfaces, they must have kind "type", which means they must not be higher-kinded, they must be fully "applied"

func (l *List[T]) Append[T](t T) {
    *l = append(*l, t)
}

tyvars don't have to textually match for type abstractions to be equal, use unification to check equivalence

unapplied type abstractions at the value level can be passed around like normal values, and have func[...](...) (...) types

methods with tyvars are normal funcs with tyvars, List.Append (as above) has type func[T](List[T], T), and myList.Append has type func(Z), where Z is whatever the type arg for myList is

<!-- variable/value types must have kind "type" --> // not true for ty abs values!

type Lister interface[L] {
    L[T] Append[T](T) // Append[T](L[T], T)
}

var l Lister = List[int]{}

l.Append[int](123)

func Append[L, T](l L[T], t T)
Append // func[L, T](L[T], T)
Append[List, int] // func(List[int], int)
func Append(l interface{}, t interface{})

func Map[F, A, B](f F[A], m func(A) B) F[B] {

}

type Comparable interface[C] {
    C Equal(C) bool
    C NotEqual(C) bool
}

type Orderable interface[O Comparable] {
    O Greater(O) bool
    O GreaterEqual(O) bool
    O Less(O) bool
    O LessEqual(O) bool
    O Max(O) O
    O Min(O) O
}

type Number interface[N] {
    N Plus(N) N
    N Minus(N) N
    N Multiply(N) N
    N Negate() N
    N Abs() N
    N Sign() N
}

type Imaginary interface[I Number] {
    I Real() Real
    I Imaginary() Real
}

type Real interface[R interface{Number, Orderable}] {
    R Rational() Rational
}

type Rational interface[R Real] {
    R Numerator() Integer
    R Denominator() Integer
}

type Integer interface[I Real] {
    I Divide(I) I
    I Modulus(I) I
    I Quotient(I) I
    I Remainder(I) I
}

type Mappable interface[M] {
    M[A] Map[A, B](func(A, B)) M[B]
}

type Wrap[W, A] func(A) W[A]

type Wrapper interface[W] {

}


incoming: pointer, value
require: pointer, value

incoming pointer, require pointer
    Max func[T](T, T)
        Max[int](123, 456)
        Max[int](my123, my456)
    Greatest func[T](*T, T)
        Greatest[int](123, 456) // error
        Greatest[int](&my123, my456)
    *T Greatest[T](T)
        123.Greatest[int](456) // error
        my123.Greatest(my456)

    interface[L] {
        L[T] Append[T](T) // Append[T](L[T], T)
    }
incoming pointer, require value
incoming value, require pointer
incoming value, require value


type SliceList[E] []E

func (l *SliceList[E]) Append[E](e E) {
    *l = append(*l, e)
}

var rawSliceList SliceList = SliceList{1, 2, 3}
rawSliceList.Append(myEmptyInterface)
SliceList.Append[interface{}](rawSliceList, myEmptyInterface)

var intSliceList SliceList[int] = SliceList[int]{1, 2, 3}
intSliceList.Append[int](myInt)
SliceList.Append[int](intSliceList, myInt)

func Concat[E](as List[E], bs List[E]) List[E]
    Concat => func(as List[interface{}], bs List[interface{}]) List[interface{}]
        myNewInterfaces = Concat(myInterfaces, myOtherInterfaces)
    Concat[int] => func(as List[int], bs List[int]) List[int]
        myNewInts = Concat[int](myInts, myOtherInts)

type List[E] interface {
    Append(E)
}

var l List[int] = SliceList[int]{1, 2, 3}
l.Append(456)

type SliceList[A] []A

func (l *SliceList[A]) Append[A](a A) {
    *l = append(*l, a)
}

type List interface[L] {
    L[A] Append[A](a A)
}

var list List = SliceList[int]{1, 2, 3}
list.Append[int](4)
list.Append[string]("5") // BUG???

// There's something different between the Interface.Method(this, arg, arg) and this.Method(arg, arg) calls...where this is checked beforehand in the latter case

type List2[A] interface[L] {
    L[A] Append(a A)
}

// receiver type args must not be 

var list2 List2[int] = SliceList[int]{1, 2, 3}
list2.Append(4)

func Append[A](l SliceList[A], a A) {
    *l = append(*l, a)
}

var intSliceList SliceList[int] = SliceList[int]{1, 2, 3}
Append[int](intSliceList, "test") // error

type AssignMaxer interface[M] {
    (x *M) Max(y M)
}

type Int int

var _ AssignMaxer = Int(0)
func (x Int) Max(y Int) // doesn't compile because AssignMaxer requires pointer receiver

var _ AssignMaxer = Int(0)
func (x *Int) Max(y Int)

type ReturnMaxer interface[M] {
    (x M) Max(y M) M
}

type Int int

var _ ReturnMaxer = Int(0)
func (x Int) Max(y Int) Int // compiles

var _ ReturnMaxer = Int(0)
func (x *Int) Max(y Int) Int // compiles because pointer receiver can convert to value receiver (?)

type Iterable[A] interface {
    Iterator() Iterator[A]
}

type Iterator[A] interface {
    HasNext() bool
    Next() A
}

type Collection[A] interface[C] {
    Add(A) bool
    AddAll(Collection[A]) bool
    Clear()
    Contains(A) bool
    ContainsAll(Collection[A])
    Empty() bool
    Iteratable[A]
    Remove(A) bool
    RemoveAll(Collection[A]) bool
    RetainAll(Collection[A]) bool
    Size() int
    Slice() []A
}

type List[A] interface[L] {
    AddIndex(int, A)
    Collection[A]
    Get(int) A
    Index(A) int
    LastIndex(A) int
    RemoveIndex(int) A
    Set(int, A)
    Slice(int, int) List[A]
    Sort(func(A, A) bool)
}

// make nil the zero value for any type T, such that `var _ T = nil` and `T(nil)` work for any T

// polymorphic funcs are assignable to applied variants of themselves
var f func(int) int = func[A](A) A {}

type List3[A] interface[L] {
    L Append(A)
}

type BadList struct{}

func (BadList) Append[A](A) {}

var list List3[int] = BadList{}
list.Append(123)

---

type Int int // gets +
type Int struct { int } // does not get +

x + y => x.Plus(y), where Plus: P func(P) P

func (x int) Sum(y int) int
func (x int) Difference(y int) int
func (x int) Product(y int) int
func (x int) Quotient(y int) int
func (x int) Remainder(y int) int

type Int int

Int(x) + Int(y) // Int(x).Sum(Int(y))

// arithmetic ops correspond nicely to methods, only requirement is both operands have same time and the op results in the same type

// type variables are assignable to their constraint interfaces and interfaces those interfaces are assignable to, like interface{}

// comparison operations return an untyped boolean value!!!!! fuck! can't reproduce that with interfaces.
// comparison operands must be assignable one way or the other for some reason! so you can't do io.Reader(x) == io.Writer(x), even
// though runtime types and values are equal

func (x int) Equal(y int) bool
T Equal(T) bool

34.Equal(myInterface)

// interface values cannot have Equal methods, because then Equal would be part of the interface! e.g. interface{} has no methods,
// and yet myInterface.Equal would be defined, which is contradictory. same problem for defined types whose underlying type is an
// interface: you can't declare methods on them! e.g. type T interface { M() }; func(T) M() {} // error, what do T.M or T(x).M mean?


```