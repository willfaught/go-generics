## Type variables

```
type1
type2
a
b
```

## Product types

```
product{}

product {
    name1 type1
}

product {
    name1 type1
    name2 type2
}
```

```
a := product {
    name1 type1
    name2 type2
}{
    name1: value1,
    name2: value2,
}

type T product {
    name1 type1
    name2 type2
}

b := T{
    name1: value1,
    name2: value2,
}

return a.name1, b.name2
```

## Sum types

```
sum {
    name1
}

sum {
    name1
    name2
}

sum {
    name1 type1
}

sum {
    name1 type1
    name2 type2
}

sum {
    name1
    name2 type2
}
```

At least one case is required.
No type for a case implies an empty struct type.
The default value is the last case, or a single case prefixed with `default`.
Selectors return the case value, having the type of the value.
Optional Boolean assignments do what you would expect.

```
a := sum {
    name1
}.name1{}

type T sum {
    name1
}

b := T.name1{}

return a, b
```

```
a := sum {
    name1 type1
    name2 type2
}.name1(value1)

type T sum {
    name1 type1
    name2 type2
}

b := T.name2(value2)

return a.name1, b.name2

if name1, ok := x.name1; ok {
    // ...
}

switch c := b.(sum) {
case T.name1:
    return c.foo
```

```
type Color sum {
    Red
    White
    Blue
}

switch color {
case Color.Red:
case Color.White:
case Color.Blue:
}

type Maybe forall(A) {
    sum {
        Just A
        Nothing
    }
}

var m Maybe(int) = Maybe(int).Just(33)
```

```
sum {
    red
    white
    blue
}.red

type c sum {
    red
    white
    blue
}

return c.red, c.white, c.blue

type door sum {
    open
    closed {
        locked bool
    }
    foo {
        int
        string
        bool
        rune
    }
    foo(int, string, bool, rune)
    bar(x, y, z float64)
}

switch myDoor {
case door.bar(x, y, z):

}

return []door{door.open, door.closed(true)}
```

TODO: Add iota support for sum types? If so, make first case default since iota starts at zero.

## Type declarations

```
type1 = type2
type3 = type4
```

type1 and type3 are bound within type2 and type4.

## Parametric polymorphism

### Type abstractions

```
forall(type1 interfacetype1, type2 interfacetype2, type3 interfacetype3) {
    product {
        name1 type1
        name2 type2
        name3 type3
    }
}
```

### Type applications

```
type1(type2, type3)
```

## Ad-hoc polymorphism

### Interface types

```
interface(type1 interfacetype1) {
    (t1 type1) m1(t2 type2) (t3 type3)
    (t1 type1) m2(t2 type2) (t3 type3)
    (t1 type1) m3(t2 type2) (t3 type3)
}
```

Equivalent to this, except the type arg is supplied at runtime:

```
forall(type1 interfacetype1) {
    interface {
        (t1 type1) m1(t2 type2) (t3 type3)
        (t1 type1) m2(t2 type2) (t3 type3)
        (t1 type1) m3(t2 type2) (t3 type3)
    }
}
```

## Scratch

```
func f (int, int) int {
    return 0
}

switch x {
case (a, b, c):
case Foo(d, e):
case Bar.Baz(f, g, h):
}

func F(r io.Reader, w io.Writer) (c io.Closer, err error) {
    
}
```

## Convert predeclared types

```
map(type1, type2)
chan(type1)
slice(type1)
array(intconst1, type1)
func(tupletype1, tupletype2)
```

## 


## Examples

Grammar:

```
expr = sum {
    add product {
        left expr
        right term
    }
    subtract product {
        left expr
        right term
    }
    term term
}

term = sum {
    add product {
        left term
        right factor
    }
    subtract product {
        left term
        right factor
    }
    factor factor
}

factor = sum {
    variable string
    paren expr
    negate factor
}
```

Slice:

```
slice = forall(element interface{}) {
    sum {
        nonnil product {
            len int
            cap int
            
        }
        nil product{}
    }
}
```

Pointer:

```
pointer = forall(element interface{}) {
    sum {
        nonnil element
        nil product{}
    }
}

pointer(int) // *int
```

Map:

```
map = forall(index, element interface{}) {
    sum {
        nonnil list(pair(index, element))
        nil product{}
    }
}

map(string, int)
```

## See also

https://chadaustin.me/2015/07/sum-types/
