# Generics for Go

Let's add [generics](https://en.wikipedia.org/wiki/Generic_programming) to Go 1.17!

First, let's focus on generic functions. By function value, I mean what's specified by function and method declarations, method signatures, method expressions, method values, and function literals. They all produce a runtime function value. Methods are just functions where the first parameter is the receiver.

We currently use interface types when we want to use the same function with varying types. As a simple example, here's a function that takes two values of the same type, then returns the first one:

```go
func(x, y interface{}) interface{} { return x }
```

This approach has some downsides:

- We can't specify in the function's type that arguments must have the same type
- We can't specify in the function's type that results must have the same type as arguments
- We can't specify in the function's type that it will behave the same regardless of the argument types
- We can't specify in the function's type that all argument types are valid
- We can't use result types directly without type assertions
- We can't allocate arguments on the stack

Here's an example of all of these downsides:

```go
func First(x, y interface{}) interface{} {
    switch y.(type) {
    case string:
        return y
    default:
        return x
    }
}

var _ = First(123, "456").(int) + 789
```

We want a way to use the same code with varying types without these downsides. We can do this with a kind of code template: factor a type out of code with a variable, then substitute the variable with varying types as needed.

First, we replace the interface type with a new kind of type called a **type variable**:

```go
func(x, y $A) $A { return x }
```

A type variable is a placeholder for another type, called a **type argument**.

Now we can:

- Specify in the function's type that arguments must have the same type
- Specify in the function's type that results must have the same type as arguments

Then we wrap the function in a new kind of value called a **type abstraction**:

```go
$abs($A) func(x, y $A) $A { return x }
```

Specifically, it's a **type-to-value abstraction** or **generic value**, since types go in (type arguments) and values come out (functions). It binds the type variable with a matching **type parameter**. We can now 

By the way, a **value abstraction**, specifically a **value-to-value abstraction**, is a normal function.

Then we specify that the type arguments must implement the interface:

```go
$abs($A interface{}) func(x, y $A) $A { return x }
```

Now we can:

- Use result types directly without type assertions
- Allocate arguments on the stack (if the generics implementation chooses to do so)

Since we expect that all type arguments are valid, it's invalid to put a type variable where it would be invalid for some type argument.

For example, consider this variation of the type-to-value abstraction above, where both parameters have type `int`, and the result still has type `$A`:

```go
$abs($A interface{}) func(x, y int) $A { return x }
```

This might seem to make some kind of sense: the result type is a type variable, which seems like it should "match" any result, even if that result is always the same. This even seems to work in a straightforward case: If we substitute `int` for `$A`, we get:

```go
func(x, y int) int { return x }
```

which is well-typed and valid. However, if we substitute `string` for `$A`, we get:

```go
func(x, y int) string { return x }
```

which is badly-typed and invalid. Therefore, this particular type-to-value abstraction is also badly-typed and invalid.

To ensure that all type arguments are valid, every type variable in a result type of a function value specification (a function or method declaration, or a function literal) must have a matching counterpart in a parameter type of the same function value, or an outer function value.

Since we expect code to behave the same regardless of the type argument, we must constrain what functions can do with type variable values so that the type argument can't affect function behavior. Type conversions, type assertions, and type switches for type variable values are prohibited.

# 



Since we expect code to be valid for any type argument, and code to behave the same regardless of the type argument, a property called [parametricity](https://en.wikipedia.org/wiki/Parametricity), we don't permit:

- Conversion to or from type conversions, type assertions, or type switches. Reflection of a type variable's value only reveals information about the constraint. Type variables are only assignable to identical type variables.

To ensure , meaning the code must be valid for any type argument, and 

Now we can have type variables in parameter and result types that tie them together, we can specify their method sets without requiring them to be interface values, 

parametricity: every type argument must be valid for the code, and the code behaves the same regardless of the type argument
    tyvars in func results
    type assertions etc for tyvars

    $abs($A interface{}) func(x, y int) $A {
        var z int = nil
        if x == y {
            z = x
        } else {
            z = y
        }
        return z
    }

var choose func(interface{}, interface{}) interface{} = func(x, y interface{}) interface{} { return x }

var choose $absty($A interface{}) func(x, y $A) $A = $abs($A interface{}) func(x, y $A) $A {
    var z $A = nil
    if x == y {
        z = x
    } else {
        z = y
    }
    return z
}




$app(choose, int)(1, 2) == 2
$app(choose, string)("1", "2") == "2"

func(x, y interface{}) interface{} {
    var z interface{} = nil
    if x == y {
        z = x
    } else {
        z = y
    }
    return z
}



// tyabs

Type declarations have the same downsides.

type Node struct {
    Value interface{}
    Prev, Next *Node
}

type Node $abs($A interface{}) struct {
    Value $A
    Prev, Next *Node
}

$app(Node, int){Value: 1}
$app(Node, string){Value: "1"}





func Choose(x, y interface{}) interface{} {
    if x == y {
        return x
    }
    return y
}

var Choose2 = func(x, y interface{}) interface{} {
    if x == y {
        return x
    }
    return y
}

func Choose3 = func(x, y interface{}) interface{} {
    if x == y {
        return x
    }
    return y
}


```