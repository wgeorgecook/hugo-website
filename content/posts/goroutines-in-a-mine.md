+++
title = "Goroutines in a Mine"
date = "2024-09-22T22:02:04-04:00"
author = "William George Cook"
tags = ["go", "golang", "goroutines", "asynchronous"]
keywords = ["go", "golang", "goroutines", "asynchronous"]
description = "Asynchronous Workloads Using Goroutines"
showFullContent = false
readingTime = false
hideComments = true
+++
# A Canary? In a Gem Mine?
Goroutines are, for me, _the killer feature of Go_. Asynchronous communication between isolated, concurrent workloads was the use case that made me a Go developer. It's a complex topic, and it can be really easy to do improperly. Heck, I even had to give this little projects a few good go overs to make the example correct. 

## Concurrent work?
Concurrency is the ability of a program to do a collective piece of work in chunks. Some pieces of that work take longer than others, and sometimes the order in which work is performed does not matter. We can use Goroutines to piece out this work, letting the scheduler start and stop processes as necessary. If you would like some very Go-specific information about the language implementation of concurrency, I _highly_ recommend reading the [Concurrency section of _Effective Go_ on go.dev](https://go.dev/doc/effective_go#concurrency).

## A refresher 
This code (found in my [Github repo here](https://github.com/wgeorgecook/goroutines-in-a-mine)) utilizes interfaces in a similar way I explained in [my botanical exploration of abstract types in Go](https://williamcook.dev/posts/botanical_exploration_of_abstract_types_in_go/) blog post. There is also code generated via `go:generate` comments. Which you can learn more about [in my post about using Go Generate for Protobuf](https://williamcook.dev/posts/go_generate_protobuf/). Neither of those are required reading, but can help you understand the code a little better if some of these topics are a little foreign to you. 

## More Than One Way to Mine Some Diamonds
If you've ever spent several days mining some land under your base in _Minecraft_, then you know there's lots of different ways to get those diamonds. You can dig some nice long strips running parallel to catch as much at one level as you want. You can find a cave system and go spellunking until you happen upon a diamond vein. You can load up on TNT and blow up everything withing a couple of chunks and catch what's left over. You can group up with friends. You can take some solo time. It doesn't matter how the diamonds are found, it only matters that they are. 

## Worker Patterns in Go
Many times you want to implement the same pattern in your program. You can have a stream of work and you need to make a few different database calls to build a response. Some data from Mongo, a little bit of Postgres. Grab some data from an S3 bucket. Whatever! The pieces are entirely independent of each other, so they don't need to wait for each other to complete before another can start working again. Now, if you will excuse my brief foray into practicallity, we can resume the contriving. 

## High Ho!
Let's look at some code. 
```go
func init() {
	inChan = make(channeling.InputChan)
	errChan = make(channeling.ErrChan)
	doneChan = make(chan bool, 1)
	workChan = make(chan bool, 1)
}
```
In the `init` function, I create several channels. Channels are the way Goroutines communicate with each other. If you can tell from the context, these are channels for input workers, error reporting, to indicate when work is done, and work to do. Keep reading and see how we send data on a channel.

```go
	for _, name := range names {
		inChan <- worker.New(name, workChan)
	}
```
The syntax is pretty simple: 
```go 
chanToReceive <- thingToSend
```

Similarly, we can read from channels in a loop. Let's examine the `channeling` package. 

```go
func Process(inChan InputChan, errChan ErrChan, done DoneChan, workChan WorkChan) {
	doWork := func(i Input) {
		if err := i.DoWork(); err != nil {
			errChan <- err
		}
	}

	log.Println("Process is listening...")
	var doneCounter = 0
	for {
		select {
		case i := <-inChan:
			go doWork(i)
		case <-workChan:
			doneCounter++
			if doneCounter == 7 {
				done <- true
			}
		}
	}

}
```

To read from a channel, you can either assign it using the walrus or use it a signal. In either case, you'll want a `select` (if reading from multiple channels) or use `range` on a single channel. 

## Off to Work We Go
A keep observer noticed that `Process` function above reads from the `inChan` and assigns it to `i`. It then calls `doWork` passing `i` in as an argument. But we do something special with this function call. It's all about those two magic letters `g` and `o`. Goroutines are spawned when you call a function (either named or anonymous) using the `go` keyword. When a function is called this way, the work is passed off to the scheduler and it unblocks the function it was called from. In the instances of our hard working dwarfs, their work takes an arbitrary amount of time.
```go
func (d Dwarf) DoWork() error {
	d.logStep("doing some work!")
	defer d.sendDone()
	randTime := rand.Int() % 100
	time.Sleep(time.Duration(randTime/5) * time.Second)
	if randTime%2 == 0 {
		return d.Error
	}
	d.logStep("success!")
	return nil
}
```
Without using Goroutines, all of this sleepiness could prevent other dwarves from completing their work in a timely manner! 

## Synchronous, But With Extra Steps
It's important to test your application and ensure that you have the right code design. You could very likely be implenting a purely synchronous workflow by missing a goroutine, or by locking until one goroutine exits. On the other end, you could get deadlock by preventing your goroutines from exiting (but keeping them alive)! 

## Raise Your Own Pickaxe
There's enough code in this repo for you to see how to handle error reporting when doing work in goroutines. You can implement contexts to end work early if it's not needed anymore. Explore how you can use a `sync.WaitGroup` to handle the blocking. Thinking in asynchronous processes requires a bit of a paradigm shift. Build the program (`go build .` or `go run main.go`) and watch the work queue up and finish in a non-deterministic way. So please experiment and discover what treasures await to those willing to venture the depths of this mine 💎
