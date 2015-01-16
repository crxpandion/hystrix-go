hystrix-go
==========

[![Build Status](https://travis-ci.org/afex/hystrix-go.png?branch=master)](https://travis-ci.org/afex/hystrix-go)

[Hystrix](https://github.com/Netflix/Hystrix) is a great project from Netflix.

> Hystrix is a latency and fault tolerance library designed to isolate points of access to remote systems, services and 3rd party libraries, stop cascading failure and enable resilience in complex distributed systems where failure is inevitable.

I think the Hystrix patterns of programmer-defined fallbacks and adaptive health monitoring are good for any distributed system. Go routines and channels are great concurrency primitives, but don't directly help our application stay available during failures.

hystrix-go aims to allow Go programmers to easily build applications with similar execution semantics of the Java-based Hystrix library.

For more about how Hystrix works, refer to the [Java Hystrix wiki](https://github.com/Netflix/Hystrix/wiki)


How to use
----------

```go
import "github.com/afex/hystrix-go/hystrix"
```

### Execute code as a Hystrix command

Define your application logic which relies on external systems, passing your function to ```hystrix.Go```. When that system is healthy this will be the only thing which executes.

```go
hystrix.Go("my_command", func() error {
	// talk to other services
	return nil
}, nil)
```

### Defining fallback behavior

If you want code to execute during a service outage, pass in a second function to ```hystrix.Go```. Ideally, the logic here will allow your application to gracefully handle external services being unavailable.

This triggers when your code returns an error, or whenever it is unable to complete based on a [variety of health checks](https://github.com/Netflix/Hystrix/wiki/How-it-Works).

```go
hystrix.Go("my_command", func() error {
	// talk to other services
	return nil
}, func(err error) error {
	// do this when services are down
	return nil
})
```

### Waiting for output

Calling ```hystrix.Go``` is like launching a goroutine, except you receive a channel of errors you can choose to monitor.

```go
output := make(chan bool, 1)
errors := hystrix.Go("my_command", func() error {
	// talk to other services
	output <- true
	return nil
}, nil)

select {
case out := <-output:
	// success
case err := <-errors:
	// failure
}
```

### Enable dashboard metrics

In your main.go, register the event stream HTTP handler on a port and launch it in a goroutine.  Once you configure turbine for your [Hystrix Dashboard](https://github.com/Netflix/Hystrix/tree/master/hystrix-dashboard) to start streaming events, your commands will automatically begin appearing.

```go
hystrixStreamHandler := hystrix.NewStreamHandler()
hystrixStreamHandler.Start()
go http.ListenAndServe(net.JoinHostPort("", "81"), hystrixStreamHandler)
```

### Setting Timeouts and Concurrency Levels

By default hystric-go will error out your request if the request takes more than 10 seconds or there are more than 10 outstanding requests for a given command. To change this set the timeout and concurrency level for your command.

```
hystrix.SetConcurrency("my_command", 50)
hystrix.SetTimeout("my_command", 100 * time.Millisecond)
```

Build and Test
--------------

- Install vagrant and VirtualBox
- Clone the hystrix-go repository
- Inside the hystrix-go directory, run ```vagrant up```, then ```vagrant ssh```
- ```cd /go/src/github.com/afex/hystrix-go```
- ```go test ./...```
