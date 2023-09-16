# [Golang]: map[string]interface{} in Go

## Intro

I'm sure no matter what level you are when it comes to using Go, you would see the code snippet `map[string]interface{}` so often that you might even wonder if it's some kind of syntax sugar in Go. Well it's not.

So what is a `map[string]interface{}` in Go, and why is it so useful? If you have the same confusion like I did, let's find out by reading this article!

## Golang map[string]interface{} example

Let's start with a simple example:

```go
foods := map[string]interface{} {
    "bacon": "delicious",
    "eggs": struct {
        source string
        price float64
    }{"chicken", 0.85},
    "steak": true,
}
```

Assume you have knowledge of the `map` type in Go, you should have no problem reading this code. The type of `foods` variable is a map where the keys are strings and the values are of type `interfaces`.

So what's that? Go `interfaces` are worthy of a tutorial series in themselves, though it’s one of those topics that seems a lot more complicated than it actually is; it’s just a little unfamiliar to most of us at first.

Suffice it to say here that an `interface` is a way of referring to a value without specifying its type. Instead, the `interface` specifies what methods it has.

So what is `interface{}`? Pronounced ‘empty interface’, it’s the interface that specifies no methods at all! Note that this doesn’t mean that `interface{}` values must have no methods; it simply doesn’t say anything at all about what methods they may or may not have.

In the words of a Go proverb, `interface{}` says nothing.

## map[string]any in Go

So what data type would satisfy an empty interface? Well, basically any. Because `interface{}` puts no constraints at all on the values it accepts, any type is okay. That’s why Go recently added the predeclared identifier `any`, as a synonym for `interface{}`.

When you need to store a collection of arbitrary values of any type, then, identified by strings, a `map[string]interface{}` or `map[string]any` is the ideal choice and of best practice in Go.

## Why is interface{} so useful?

What’s the point of `interface{}`, then, if it doesn’t tell us anything about the value? Well, that’s precisely why it’s useful: it can refer to anything! The type `interface{}` (or, as we’d now say, `any`) applies to any value.

A variable declared as `interface{}` can hold a string value, an integer, any kind of struct, a pointer to an os.File, or just anything you can think of.

Suppose we need to write a function that prints out the value passed to it, but we don’t know in advance what type this value would be. This is a job for the empty interface:

```go
func printAnything(v any)
```

Actually, if you read the source code of the famous `fmt` package, you would notice the signature of `Println`:

```go
func Println(a ...any) { ... }
```

## map[string]any and arbitrary data

Similarly, if we want a collection of different kinds of thing, each one identified by a string, which is a convenient way to organize arbitrary data, we can do that with `map[string]interface{}`. In fact, we just described the schema of JSON objects, for example. Take this raw JSON data:

```json
{
  "name": "John",
  "age": 29,
  "hobbies": ["martial arts", "breakfast foods", "piano"]
}
```

We can see that this is a collection of things identified by string keys, but what kind of things? We have a string, an integer, and an array of strings.

Supposing we needed to translate this into a Go struct value, we could define a type like this:

```go
type Person struct {
    Name    string
    Age     int
    Hobbies []string
}
```

Great. But this requires that we know the schema of the object in advance. What if someone gives us arbitrary JSON data, and we need to unmarshal it into a Go value? How can we possibly do that, given that all we know is that it’s a map of strings to objects of any type?

## Decoding JSON data to map[string]any

Suppose that we have the biographically questionable JSON data about me stored in a variable called `data`. How can we unmarshal this into a Go variable so that we can start looking at it? What type would that variable need to be?

```go
p := map[string]any{}
err := json.Unmarshal(data, &p)
// check error
```

Provided there are no errors, the `p` variable now contains our arbitrary data.

But, given that we know nothing at all about the type of each value in the map, what can we usefully do with it?

## Using map[string]any data

One thing we can do is use a type switch to do different things depending on the type of the value. Here’s an example:

```go
for k, v := range p {
    switch c := v.(type) {
    case string:
        fmt.Printf("Item %q is a string, containing %q\n", k, c)
    case float64:
        fmt.Printf("Looks like item %q is a number, specifically %f\n", k, c)
    default:
        fmt.Printf("Not sure what type item %q is, but I think it might be %T\n", k, c)
    }
}
```

The special syntax `switch c := v.(type)` tells us that this is a type switch, meaning that Go will try to match the type of `v` to each case in the switch statement. For example, the first case will be executed if `v` is a string:

```
Item "name" is a string, containing "John"
```

## When to use map[string]interface{}

As we’ve seen, the “map of string to empty interface” type is very useful when we need to deal with data that comes from outside the Go world; for example, arbitrary JSON data of unknown schema. Many web APIs return data like this, for example.

It’s also extremely common when writing Terraform providers, which makes sense; Terraform resources are also essentially maps of strings to arbitrary data. It’s recursive, too; the ‘arbitrary data’ is also often a map of strings to more arbitrary data. It’s `map[string]interface{}` all the way down!

Configuration files, too, generally have this kind of schema. You can think of YAML files as being maps of string to empty interface, just like JSON. So when we’re dealing with structured data of any kind, we’ll often use this type in Go programs.

## When NOT to use map[string]interface{}

A `map[string]any` is like one of those universal travel adapters, that plugs into any kind of socket and works with any voltage. You can use it to protect your own vulnerable programs from damage caused by weird, alien data.

Should you use `map[string]interface{}` values within your own programs, when there’s no need to handle arbitrary input data? No, you shouldn’t. While it might seem convenient to not have to explicitly define the schema of your objects, that can lead to all kinds of problems.

For one thing, since `interface{}` intrinsically says nothing, whenever we deal with a value of this type, we have to use protective type assertions to prevent panics:

```go
if _, ok := x.(string); !ok {
    log.Fatal("Type assertion failed!")
}
```

In other words, it’s a lot more difficult to write safe, reliable programs that operate on such maps. If your library produces data of this kind, it will hardly endear you to users. Instead, just use a plain old struct, which enables compile-time type checking and is much more convenient to deal with.

Just like a travel adapter, `map[string]any` is a bit awkward to use when you’re at home and you can rely on your sockets all having the expected voltage and pin schema.

But when you’re in contact with other worlds, outside the Go’s type system, `map[string]any` is perhaps the ultimate travel accessory. Use it well!
