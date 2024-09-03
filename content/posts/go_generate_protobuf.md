+++
title = "Go_generate_protobuf"
date = "2024-09-03T14:30:59-04:00"
author = "William George Cook"
tags = ["go", "golang", "build tools", "toolchain", "protocol buffers", "grpc"]
keywords = ["go", "golang", "protobuf", "grpc", "protoc", "proto"]
description = "Using Go Generate to Compile Proto Files"
showFullContent = false
readingTime = false
hideComments = true
+++

# Playground for Go Generate Using Proto Files
This is a small application to demonstrate how to use `go generate` to compile proto files into Go code. 

Note: all code is available on GitHub [here](https://github.com/wgeorgecook/playgen).

## Go Generate
The [original blog post](https://go.dev/blog/generate) by Rob Pike is well worth a read. 
At the bottom, Rob says 
> Please use go generate creatively. It’s there to encourage experimentation.

This is one such experimentation. 

## Protobuf, gPRC, and Generating Code
If you are unfamiliar with [Protocol Buffers](https://protobuf.dev/) and [gRPC](https://grpc.io/docs/languages/go/quickstart/), those linked resources are worth a read. However, just know that Protobuf is a language spec that allows you to define abstract API contracts within generated code stubs. Your application needs to implement these interfaces for clients to consume. The benefit is that both client and server consume and fulfill requests using the same base `.proto` file and its generated code, allowing for incredible type safety. 

## Protoc Hassles
Go developers using Protobuf for a few years are probably familiar with the `protoc` binary used and the `protoc-gen-go` plugin. This is the tool we use to generate Go stubs from proto files. Protoc commands can be provided via scripts, in Dockerfiles, or ran directly to ensure that code is generated and types are accessible to your application. This decouples the proto compilation step from your app compilation. If you want to couple the two together, `go generate` may be your answer. 

# Go Generate (again)
The `go generate` command searches through your Go files for the `//go:generate` comment (notice no space between the slashes and `go`). Adding this tag tells the Go toolchain to run whatever shell commends follow. 

## Tagging for Go Generate 
You can see here that `main.go` has two go generate tags. 
```
//go:generate protoc --go_out=. --go-grpc_out=. proto/helloapis/messages.proto
//go:generate protoc --go_out=. --go-grpc_out=. proto/helloapis/services.proto
```

These two tags will compile protos found in `proto/helloapis` and move them into the directory indicated by their package name. In this instance, that directory is `gen/helloapis/v1`. 

# Building the Application
Attemping to run `go build .` before compiling the protos will fail. 
> ❯ go build .
client/client.go:6:2: package playgen/gen/helloapis/v1 is not in std (/snap/go/10630/src/playgen/gen/helloapis/v1)

To resolve this, run `go generate ./...`. 
> ❯ go generate ./...

Notice that a new `gen` directory is in the repo root. 
```
❯ ls
build  client  docker-compose.yaml  gen  go.mod  go.sum  main.go  proto  README.md  server
```

The application is now buildable and runnable!
```
❯ go build .
❯ ls
build  client  docker-compose.yaml  gen  go.mod  go.sum  main.go  playgen  proto  README.md  server
❯ ./playgen
Hello, vim-go!
```