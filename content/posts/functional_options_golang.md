+++
title = "Functional_options_golang"
date = "2024-08-07T21:08:38-04:00"
author = "William George Cook"
tags = ["go", "design pattern", "extensible api"]
keywords = ["go", "golang", "function", "arguments", "design patterns", "variadic options in Golang"]
description = "A primer of the functional option pattern in Go"
showFullContent = false
readingTime = false
hideComments = true
+++

# Lets say...
You are the maintainer of a widely used function in a fairly large service. This function abstracts away 
all of the boilerplate necessary to log an error. The wider service has several different debugging tools. 
For example, it could:
1. Log errors to [Rollbar](https://rollbar.com)
2. Instrument [OpenTelemetry](https://opentelemetry.io/)
3. Support logging to stdout, to disk, or an async log aggregator.

the list can theoretically go on and on. 

## LogErrors's Humble Beginnings
To start, you write your function to support your own use case, and make the api easy to consume for others. 

```code
func LogError(e error, msg string) {
    // assume a logger is configured
    logger.Error(e.Error + ": " + msg)
}
```
This is a simple enough function that formats an error and invokes the configured logger at the Error level. 

## LogError's Organic Growth
As this service matures, things can get out of hand. Let's introduce some other, optional dependencies in a naive way. 
```code
func LogError(e error, msg string, shouldRollbar bool, spanCtx context.Context) {
    fmtErr := e.Error + ": " + msg
    if shouldRollbar {
        rollbar.Error(fmtErr)
    }
    if spanCtx != nil {
        span := otel.NewSpan("LogError", spanCtx)
        defer span.End()
        span.RecordError(fmtErr)
    }
    logger.Error(fmtErr)
}
```

This is fine and functional, but you can see how adding additional, optional features requires a full refactor of every function call. Furthermore, the caller has to populate these arguments even if they are falsy or `nil`. 

## A Better Way
We can enforce some required arguments by defining them as we would before. Let's say that `err` and `msg` are required params. However, logging to a span or triggering a Rollbar are completely optional. We can save the caller a hastle by not requiring them in the function call at all by using the functional options pattern. 

# Functional Options
We need to create an `option` type that we use internally to drive the logic for behavior. Leave this unexported so the caller doesn't try and use it directly.
```code
type option struct {
    withRollbar bool
    withContext context.Context
}
```

The real magic happens when we create a type that returns a function that accepts the option as an argument.
```code 
type Options func(*option)
```
We now create functions that return the `Options` type that the caller can use. 

```code
func WithRollbar() Options {
    return func(o *option) {
        o.withRollbar = true
    }
}

func WithContext(ctx context.Context) Option {
    return func(o *option) {
        o.withContext = ctx
    }
}
```
## Leveraging Functional Options
All of the pieces are coming together, but as `LogError` stands, it does not use options. Let's fix that by first changing the function signature.
```code
func LogError(err error, msg string, opts ...Options) {
 // function body
}
```
Now the `err` and `message` arguments are required by the compiler (even if nil or empty). The `...Options` syntax indicates that `LogError` is now a _variadic function_, and that the trailing arguments is a slice of length >= 0. The compiler will not enforce that any argument is present for the variadic argument. Functionally this means that `opts` is entirely optional! However, we still need to leverage these options within the function. 
```code
func LogError(err error, msg string, opts ...Options) {

    fmtErr := err.Error() + ": " + msg

    // instantiate an option struct to populate with 
    // provided options
    o := options{}

    // iterate through the provided options to populate
    // data on the struct
    for _, opt := opts {
        opt(o)
    }

    // start performing logic based on provided options
    if o.withRollbar {
        rollbar.Error(fmtErr)
    }

    if o.withContext != nil {
        span := otel.NewSpan("LogError", spanCtx)
        defer span.End()
        span.RecordError(fmtErr)
    }

    logger.Error(fmtErr)
}
```

You can see the function body doesn't change _all that much_. But we have significatly reduced the signature and uncomplicated the things the caller needs to worry about. 

## Using LogError
With functional arguments, all of these are valid invocations of `LogError`. 

```code
err := errors.New("there was an error!")
msg := "could not perform operation"
ctx := context.Background()
LogError(err, msg)
LogError(err, msg, WithContext(ctx))
LogError(err, msg, WithRollbar())
LogError(err, msg, WithContext(ctx), WithRollbar())
LogError(err, msg, WithRollbar(), WithContext(ctx))
```

Consumers of `LogError` now no longer need to worry about providing a nil context or falsy `shouldRollbar` value if they are unconcerned about those features. 

## Extending LogError
Now that `LogError` is a variadic function, we can add more options without needing to refactor any of its consumers. Let's implement optionally logging to disk instead of standard out. 

```code
type option struct {
    withRollbar bool
    withContext context.Context
    withLogToDisk bool // new field
}

func WithLogToDisk() Option {
    return func(o *option) {
        o.withLogToDisk = true
    }
} 

func LogError(err error, msg string, opts ...Options) {

    fmtErr := err.Error() + ": " + msg

    // instantiate an option struct to populate with 
    // provided options
    o := options{}

    // iterate through the provided options to populate
    // data on the struct
    for _, opt := opts {
        opt(o)
    }

    // start performing logic based on provided options
    if o.withRollbar {
        rollbar.Error(fmtErr)
    }

    if o.withContext != nil {
        span := otel.NewSpan("LogError", spanCtx)
        defer span.End()
        span.RecordError(fmtErr)
    }

    if o.withLogToDisk {
        // helper function assumed to exist
        logErrorToDisk(fmtErr)
        return 
    } 

    logger.Error(fmtErr)

}
```

`LogError` is now more functional for consumers who wish to leverage this option. However, because of the variadic nature of `LogError`, we don't need to do a single bit of refactoring the consumers and the api is unchanged. Anyone who wishes to log to disk instead of to standard out can do so like this:
```code
err := errors.New("there was an error!")
msg := "could not perform operation"
LogError(err, msg, WithLogToDisk())
```

# That's Pretty Neat
There is the overhead of planning for which arguments are required and which are options when maintaining a function such as `LogError`. However, if you know that the function will be broadly consumed from users with differing needs then utilizing functional options is a great way to implement those dependencies without forcing them on every consumer. Functional options allow for easily extending functionality without requiring large refactors for the consumers of its api. 