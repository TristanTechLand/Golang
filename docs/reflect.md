# [Golang pkg]: Deep Dive of Reflection in Go

## Outline

- [Intro](#intro)
- [Interface](#interface)
- [Reflect basics](#reflect-basics)
- [Reflect usages](#reflect-usages)
  - [Penetrate data composition](#penetrate-data-composition)
- [References](#references)

## Intro

To begin, let's talk about what exactly `Reflect` is. `Reflect` is a mechanism, allowing you to overlook the composition of structures to manipulate underlying values without knowing types explicitly during compile time. By using `Reflect`, we are able to write code that may deal with arbitrary types. One famous example is `fmt.Println()`, allowing us to print out any types, even self defined types.

It is generally not recommended to use `Reflect`, because it does result in multiple problems, such as lower performance, make code hard to read, postpone type bugs that can be detected during compile time till runtime in the form of `panic`, etc. However, I still consider it important to understand `Reflect` for two reasons:

1. Many standard libraries and third party packages use `Reflect`. Although you may argue that APIs are exposed to developers, but if you really want to dive deeper into source code level, or even a more deeper system level, `Reflect` would be inevitable.

2. You never know what kind of requirements would pop up! If you need to create a function / method that handles arbitrary types, `Reflect` would be the ideal choice.

Luckily, Go provided a standard library `Reflect`.

## Interface

`Reflect` is built on top of the type system, and closely related with interface.

As a refresher, let's walk through `interface` real fast. In Go, interfaces basically defines a set of functions, any type variable that implements the set of functions can be assigned to a variable with the type interface.

```go
type Animal interface {
  Speak()
}

type Cat struct {
}

func (c Cat) Speak() {
  fmt.Println("Meow")
}

type Dog struct {
}

func (d Dog) Speak() {
  fmt.Println("Bark")
}

func main() {
  var a Animal

  a = Cat{}
  a.Speak()

  a = Dog{}
  a.Speak()
}
```

In the code above, we defined an `Animal` interface with one function `Speak()`. Then we defined two structs, `Cat` and `Dog`, which both implemented the `Speak` function. By doing so, we are allowed to assign a `Dog` / `Cat` typed object to an `Animal` typed variable.

An interface variable is composed of 2 parts: type and value, `(T, V)`. If we do know explicitly the type of `T`, we can also use type assertion to obtain the underling value of the interface variable:

```go
type Animal interface {
    Speak()
}

type Cat struct {
    Name string
}

func (c Cat) SpeaK() {
    fmt.Println("meow")
}

func main() {
    var a Animal

    a = Cat{Name: "Lucca"}
    a.Speak()

    c := a.(Cat)
    fmt.Println(c.Name)
}
```

In the code above, since we know in advance that `a` holds an object of type `Cat`, we can use `a.(Cat)` as type assertion to obtain the underlying value `a` directly.

However, an important point to notice is, if the type that you're asserting does not match with the actual type, the program will panic directly. As a result when in development, the better practice is to use another form of type assertion: `c, ok := a.(Cat)`.

```go
t, ok := i.(T)

if ok {
    // continue program with t
} else {
    // fmt.Println("type assertion failed without panic!")
}
```

If `i` holds a `T`, then `t` will be the underlying value and `ok` will be true.

If not, `ok` will be false and t will be the zero value of type `T`, and no panic occurs.

Through interface, we can only call the methods defined within. Of course, we can also assert it to another interface, then we can also call the methods in the asserted interface. The premise is that the source object is required to implement the interface. I'm sure no one understands this nonsense, so let's just take a look at an example.

```go
var r io.Reader
r = new(bytes.Buffer)
w = r.(io.Writer)
```

`io.Reader` and `io.Writer` are 2 interfaces that are used most frequently in standard libraries:

```go
/ src/io/io.go
type Reader interface {
  Read(p []byte) (n int, err error)
}

type Writer interface {
  Write(p []byte) (n int, err error)
}
```

Since `bytes.Buffer` implemeted both interfaces, we can assign a `bytes.Buffer` object to `r`, which is type `io.Reader`. Then we can assert `r` to be `io.Writer`, because the source object `r`, originally type `io.Reader`, now a `bytes.Buffer` object also implements interface `io.Writer`.

From the example above, we can arrive at a generic conclusion. If an interface `A` contains all methods in interface `B`, then an interface variable of type `A` can be assigned to an interface variable of type `B` directly, because the value that interface variable of type `A` holds is guaranteed to have implemented all methods of an interface variable of type `B`. For example:

```go
// src/io/io.go
type ReadCloser interface {
  Reader
  Closer
}
```

Any variable of type `ReadCloser` can be assigned directly to a interface variable of type `io.Reader`.

An empty interface `interface{}` is a special interface. It does not define any methods. In other words, any type can be assigned to a variable of type `interface{}`.

To end this section with one point very important. Regardless of assignments or assertions between interface variables, the inner `(T, V)` is constant. The only difference is that you could call different methods through different interfaces.

## Reflect basics

In Go, `reflect` library defined an interface `reflect.Type` and a struct `reflect.Value`, which defined a sort of methods used to obtain type information, set values etc. In fact, only type descriptors implemented `reflect.Type`, but because the descriptors aren't exported, we can only use `reflect.TypeOf()` to obtain the value of `reflect.Type`.

```go
import (
  "fmt"
  "reflect"
)

type Cat struct {
  Name string
}

func main() {
  var f float64 = 3.5
  t1 := reflect.TypeOf(f)
  fmt.Println(t1.String())

  c := Cat{Name: "kitty"}
  t2 := reflect.TypeOf(c)
  fmt.Println(t2.String())
}
```

- output

```
float64
main.Cat
```

Go is a static type language, so the type of each variable is confirmed and known during compile time. Static types cannot be changed after declaration. For example, an interface variable, its static type is type of `interface` assigned. Although technically we can assign different values to it, only the inner dynamic type and value are altered. In other words, its static type remains unchanged.

`reflect.Typeof()` is used to retrieve the dynamic part of interfaces, returning in the form of `reflect.Type`. But wait a minute, he above code does not seem to have an interface type variable?

Well, let's take a look at the definition of `reflect.TypeOf()`:

```go
// src/reflect/type.go
func TypeOf(i interface{}) Type {
	eface := *(*emptyInterface)(unsafe.Pointer(&i))
	return toType(eface.typ)
}
```

`TypeOf()` accepts one argument of type `interface{}`. So the variable of type `float64` and `Cat` will firstly be casted into `interface{}`.

Correspondingly, `reflect.ValueOf()` retrieves the value of the casted `interface{}` variable, returning in the form of `reflect.Value`.

```go
v1 := reflect.ValueOf(f)
fmt.Println(v1)
fmt.Println(v1.String())

v2 := reflect.ValueOf(c)
fmt.Println(v2)
fmt.Println(v2.String())
```

- output

```
3.5
<float64 Value>
{kitty}
<main.Cat Value>
```

Since it is very common to get the type of variables, `fmt` provided a formatter `%T` to print out types:

```go
fmt.Printf("%T\n", 3) // int
```

In Go, there are an infinite number of types since we can define new types through `types`. However, there is still a limited number of primitive types. `reflect` library enumerated all primitive types in Go:

```go
/*
 * These data structures are known to the compiler (../cmd/compile/internal/reflectdata/reflect.go).
 * A few are known to ../runtime/type.go to convey to debuggers.
 * They are also known to ../runtime/type.go.
 */

// A Kind represents the specific kind of type that a Type represents.
// The zero Kind is not a valid kind.
type Kind uint

const (
	Invalid Kind = iota
	Bool
	Int
	Int8
	Int16
	Int32
	Int64
	Uint
	Uint8
	Uint16
	Uint32
	Uint64
	Uintptr
	Float32
	Float64
	Complex64
	Complex128
	Array
	Chan
	Func
	Interface
	Map
	Pointer
	Slice
	String
	Struct
	UnsafePointer
)
```

There are 26 different types. Any type can only be composed of these types.

```go
type MyInt int

func main() {
	var i int
	var j MyInt

	i = int(j)

	ti := reflect.TypeOf(i)
	tj := reflect.TypeOf(j)

	fmt.Println("type of i: ", ti.String())
	fmt.Println("type of j: ", tj.String())
	fmt.Println("kind of i: ", ti.Kind())
	fmt.Println("kind of i: ", ti.Kind())
}
```

- output

```
type of i:  int
type of j:  main.MyInt
kind of i:  int
kind of i:  int
```

The static types of `i` and `j` are `int` and `MyInt` respectively. Although the underlying type of `MyInt` is `int`, you still need to cast it explicitly to assign `j` to `i`. Notice they have the same `Kind`.

## Reflect usages

There are loads of APIs that `reflect` provides, so let's go through them with actual examples.

### Penetrate data composition

In order to see through, or say penetrate the composition of `struct`, we may need the following
functions:

- `func ValueOf(i any) Value`: returns a new Value initialized to the concrete value stored in the interface i. `ValueOf(nil)` returns the zero Value.

- `func (v Value) NumField() int`: returns the number of fields in the struct v. It panics if v's Kind is not Struct.

- `func (v Value) Field(i int) Value`: returns the object of the ith field of the reflect object

- `func (v Value) Kind() Kind`: Kind returns v's Kind. If v is the zero Value (IsValid returns false), Kind returns Invalid.

- `reflect.Int()/reflect.Uint()/reflect.String()/reflect.Bool()`

```go
type User struct {
	Name    string
	Age     int
	Married bool
}

func inspectStruct(u interface{}) {
	v := reflect.ValueOf(u)
	for i := 0; i < v.NumField(); i++ {
		field := v.Field(i)
		switch field.Kind() {
		case reflect.Int, reflect.Int8, reflect.Int16, reflect.Int32, reflect.Int64:
			fmt.Printf("field:%d type:%s value:%d\n", i, field.Type().Name(), field.Int())

		case reflect.Uint, reflect.Uint8, reflect.Uint16, reflect.Uint32, reflect.Uint64:
			fmt.Printf("field:%d type:%s value:%d\n", i, field.Type().Name(), field.Uint())

		case reflect.Bool:
			fmt.Printf("field:%d type:%s value:%t\n", i, field.Type().Name(), field.Bool())

		case reflect.String:
			fmt.Printf("field:%d type:%s value:%q\n", i, field.Type().Name(), field.String())

		default:
			fmt.Printf("field:%d unhandled kind:%s\n", i, field.Kind())
		}
	}
}

func main() {
	u := User{
		Name:    "cc",
		Age:     23,
		Married: true,
	}

	inspectStruct(u)
}
```

By using `NumField()` and `Field()` of `reflect.Value` we are able to iterate through every field of a struct, then we can perform respective operations based on its `Kind`.

One thing to notice is that some functions are only callable to specific types of the caller. `NumField()` and `Field()` can only be called by `struct`, or else `panic` occurs.

## References

| Ref             | Link                                                                                                                       |
| :-------------- | :------------------------------------------------------------------------------------------------------------------------- |
| Type assertions | https://go.dev/tour/methods/15                                                                                             |
| Go source code  | https://github.com/golang/go/blob/master/src/runtime/type.go https://github.com/golang/go/blob/master/src/reflect/value.go |
