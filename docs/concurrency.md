# [Golang]: goroutines, channels and concurrency

## Outline

- [Intro](#intro)
- [Channels](#channels)
- [Buffered Channels](#buffered-channels)
- [Anonymous functions as goroutines](#anonymous-functions-as-goroutines)
- [Channels between goroutines](#channels-between-goroutines)
- [Range and Close](#range-and-close)
- [Select](#select)
- [Mutual exclusion](#mutual-exclusion)
- [Conclusion](#conclusion)

## Intro

A few weeks ago I started as an intern working on a microservice based cloud IoT system built in Go. This is my first time working with Go so a lot of the concepts and features of Go actually amazed me in multiple ways.

In this article, I want to talk about one of the best features in Go, in my humble opinion: Goroutines and Channels.

`Goroutines` and `Channels` are a lightweight built-in feature for managing concurrency and communication between several functions executing simultaneously. This way, one can write code that executes outside of the main program so it doesn’t interrupt it and returns a value or values (or nothing if it’s just an independent operation). Go has two keywords for this: `go` and `chan`.

The implementation is surprisingly simple. First off, define a function that you wish to execute concurrently:

```go
func timesThree(num int) {
    fmt.Println(num * 3)
}
```

As you can see, it's just a regular function. Now we execute is with the keyword `go`:

```go
func timesThree(number int) {
	fmt.Println(number * 3)
}

func main() {
	fmt.Println("We are executing a goroutine")
	go timesThree(3)
	fmt.Println("Done!")
	time.Sleep(time.Second) // This sleep prevents the program for exiting
				            // before the goroutine starts
}
```

- output:

```
We are executing a goroutine
Done!
9
Process finished with the exit code 0
```

The main program creates a new goroutine for executing `timesThree` function and continues the next instruction. Therefore `fmt.Println("Done!")` is executed before the goroutine.

But what happens if we need some value returning from that function to continue with our main flow? To do this, we have to introduce another great feature of Go: `channels`.

## Channels

As the name suggests, `channels` are like two-ways streets for our data between goroutines. We have to initialize it with the function `make`, the keyword `chan` and the data type between parenthesis.

```go
ch := make(chan dataType)
```

Let’s assume we need the result of the operation. Then we need to pass the channel as a parameter to the goroutine function so it returns the result with the characters `<- `for assigning the value.

```go
func timesThree(number int, ch chan int) {
	result := number * 3
	fmt.Println(result)
	ch <- result
}

func main() {
	fmt.Println("We are executing a goroutine")
	ch := make(chan int)
	go timesThree(3, ch)
	result := <- ch
	fmt.Printf("The result is: %v", result)
}
```

- output

```
We are executing a goroutine
9
The result is: 9
Process finished with the exit code 0
```

Once the main program executes the goroutines, it waits for the channel to get some data before continuing, therefore `fmt.Println("The result is: %v", result)` is executed after the goroutine returns the result. This doesn’t mean that the main program will wait for the full goroutine to execute, just until the data is served to the channel.

Now, what if we need the goroutine to return multiple values? That’s why we have buffered channels.

## Buffered Channels

Let’s make our `timesThree` function receive an array of number and iterate over it multiplying every element by three.

```go
func timesThree(arr []int, ch chan int) {
	for _, elem := range arr {
		ch <- elem * 3
	}
}

func main() {
	fmt.Println("We are executing a goroutine")
	arr := []int{2,3,4}
	ch := make(chan int)
	go timesThree(arr, ch)
    time.Sleep(time.Second) // we let the goroutine return all the values
	result := <- ch
	fmt.Printf("The result is: %v", result)
}
```

- output

```
We are executing a goroutine
The result is: 6
Process finished with the exit code 0
```

Why didn’t the main program print all 3 values? Well, because the channel only has space for one value. We can implement a buffered channel by assigning the channel capacity by passing a second parameter to the make function with the number of elements it can get before it’s read.

```go
func timesThree(arr []int, ch chan int) {
	for _, elem := range arr {
		ch <- elem * 3
	}
}

func main() {
	fmt.Println("We are executing a goroutine")
	arr := []int{2,3,4}
	ch := make(chan int, len(arr))
	go timesThree(arr, ch)
	for i := 0; i < len(arr); i++ {
		fmt.Printf("Result: %v \n", <- ch)
	}
}
```

This way we can get all three values returned by `timesThree` function.

```
We are executing a goroutine
Result: 6
Result: 9
Result: 12
Process finished with the exit code 0
```

## Anonymous functions as goroutines

Another great feature is the possibility to execute an anonymous function as a goroutine, if we won’t reuse it. Note that we declare the function after the keyword `go` and we pass the parameters between parenthesis after the final curly brackets.

```go
func main() {
	fmt.Println("We are executing a goroutine")
	arr := []int{2,3,4}
	ch := make(chan int, len(arr))
	go func(arr []int, ch chan int) {
		for _, elem := range arr {
			ch <- elem * 3
		}
	}(arr, ch)
	for i := 0; i < len(arr); i++ {
		fmt.Printf("Result: %v \n", <- ch)
	}
}
```

## Channels between goroutines

Channels not only work for interactions between a goroutine and the main programs, they also provide a way to communicate between different goroutines. For example, let’s create a function that subtracts 3 to every result returned by `timesThree` but only if it’s an even number.

```go
func timesThree(arr []int, ch chan int) {
	minusCh := make(chan int, 3)
	for _, elem := range arr {
		value := elem * 3
		if value % 2 == 0 {
			go minusThree(value, minusCh)
			value = <- minusCh
		}
		ch <- value
	}
}

func minusThree(number int, ch chan int) {
	ch <- number - 3
	fmt.Println("The functions continues after returning the result")
}

func main() {
	fmt.Println("We are executing a goroutine")
	arr := []int{2,3,4}
	ch := make(chan int, len(arr))
	go timesThree(arr, ch)
	for i := 0; i < len(arr); i++ {
		fmt.Printf("Result: %v \n", <- ch)
	}
}
```

- output

```
We are executing a goroutine
The functions continues after returning the result
The functions continues after returning the result
Result: 3
Result: 9
Result: 9
```

Even if in this case it wasn’t necessary to have `minusThree` behave as a goroutine and have the result returned via the channel, it illustrates how the interaction between goroutines works. This is particularly useful when you have two different functions in a solution that needs to be performant and some condition of one affects the outcome of the other.

## Range and Close

These features enable us to receive continuous elements from a goroutine until it closes the channel. With the instruction `for i := range ch` we can iterate over the goroutine’s results as soon as they are sent. The goroutine should close the channel with the function close once it finishes sending data.

```go
func timesThree(arr []int, ch chan int) {
  for _, elem := range arr {
    ch <- elem * 3
  }
  close(ch)
}

func main() {
	fmt.Println("We are executing a goroutine")
	arr := []int{2,3,4,5,6}
	ch := make(chan int, len(arr))
	go timesThree(arr, ch)
    for i := range ch {
        fmt.Printf("Result: %v \n", i)
    }
}
```

If the goroutine doesn’t close the channel once it finishes sending data, the program will crash with the following error:

```
fatal error: all goroutines are asleep - deadlock!
```

This happens because the main program tries to receive a value from the channel but there isn’t an active goroutine able to send it.

I haven’t mentioned the `close` function before because as “The Go programming language” book states:

**_You needn’t close every channel when you’ve finished with it. It’s only necessary to close a channel when it is important to tell the receiving goroutines that all data have been sent. A channel that the garbage collector determines to be unreachable will have its resources reclaimed whether or not it is closed._**

## Select

How can we read from multiple channels at the same time? Works as a way to wait for multiple channels at the same time, preventing one from blocking another.

```go
func timesThree(arr []int, ch chan int) {
	for _, elem := range arr {
		ch <- elem * 3
	}
}

func minusThree(arr []int, ch chan int) {
	for _, elem := range arr {
		ch <- elem - 3
	}
}

func main() {
	fmt.Println("We are executing a goroutine")
	arr := []int{2,3,4,5,6}
	ch := make(chan int, len(arr))
	minusCh := make(chan int, len(arr))

	go timesThree(arr, ch)
	go minusThree(arr, minusCh)

	for i := 0; i < len(arr) * 2;i ++ {
		select {
			case msg1 := <- ch:
				fmt.Printf("Result timesThree: %v \n", msg1)
			case msg2 := <- minusCh:
				fmt.Printf("Result minusThree: %v \n", msg2)
			default:
				fmt.Println("Non blocking way of listening to multiple channels")
		}
	}
}
```

If we look at the console output we can see that the main program is receiving data from both goroutines at the same time.

- output

```
We are executing a goroutine
Result minusThree: -1
Result timesThree: 6
Result minusThree: 0
Result timesThree: 9
Result timesThree: 12
Result timesThree: 15
Result minusThree: 1
Result timesThree: 18
Result minusThree: 2
Result minusThree: 3
```

We can add a `default` case (like in a switch statement) if we want to execute something else every iteration that there is no data from any channel.

## Mutual exclusion

A problem that may arise when working with concurrency is when two goroutine share the same resources, which shouldn’t be accessed at the same time by multiple goroutines.

In concurrency, the block of code that modifies shared resources is called the `critical section`. Let’s illustrate with an example and what it prints on console.

```go
var n = 1

func timesThree() {
	n *= 3
	fmt.Println(n)
}

func main() {
	fmt.Println("We are executing a goroutine")
	for i := 0; i < 10; i++ {
		go timesThree()
	}
	time.Sleep(time.Second)
}
```

- output

```
We are executing a goroutine
27
81
3
9
243
6561
729
19683
59049
2187
```

Because goroutines are accessing and reassigning the same memory space at the same time, we get problematic results. In this case, `n *= 3` would be the critical section.

We can easily solve this problem by locking the block of code that access the variable by implementing mutual exclusion using `sync.Mutex`. This prevents that more than one goroutine accesses the instructions between the `Lock()` and `Unlock()` functions at the same time.

```go
import (
	"fmt"
	"time"
	"sync" // import the sync package
)

var n = 1
var mu sync.Mutex

func timesThree() {
  	mu.Lock()
	defer mu.Unlock()
	n *= 3
	fmt.Println(n)
}

func main() {
	fmt.Println("We are executing a goroutine")
	for i := 0; i < 10; i++ {
		go timesThree()
	}
	time.Sleep(time.Second)
}
```

- output

```
We are executing a goroutine
3
9
27
81
243
729
2187
6561
19683
59049
```

## Conclusion

Goroutines are an amazing and lightweight feature that makes concurrency pretty easy to implement, one of the reasons Go’s adoption hasn’t stopped to rise in the recent years (and why lately I have been loving it so much).

Thanks a lot for reading!
