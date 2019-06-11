---
layout: post
title: Generating sequence diagrams from tests in Golang
author: stein_fletcher
tags: [golang, testing, sequence diagrams]
---

![Sequence diagrams!]({{ "/images/diagram.png" }})

#### tl;dr

_Clone the <a href="https://github.com/infinityworks/music" target="_blank">example project</a> which includes a REST api and tests that render sequence diagrams - run `make test` to create the diagrams. The diagrams look like <a href="http://demo-html.apitest.dev.s3-website-eu-west-1.amazonaws.com/" target="_blank">this</a>_.

The sequence diagram generation code is now integrated into the <a href="https://apitest.dev" target="_blank">apitest.dev</a> behavioural testing library. You can can find lots of <a href="https://github.com/steinfletcher/apitest/tree/master/examples" target="_blank">examples</a> on github.com.

## Introduction

It is fairly common to see an application talk to several external services, a database and a message queue. It can be a challenge to build a mental model of the code when there are so many interconnected systems. We might choose to document these interactions by hand as a sequence diagram - but these are legacy diagrams - they become invalid the moment they are written.

In this post, we will explore a method to generate sequence diagrams from tests that describe the application interactions. We will do this in a non-invasive way - we will not change any production code. We will document an API that talks to a database and external server. We will produce a sequence diagram and a log of the HTTP and database interactions. This post focuses on the Go programming language, but it should be possible to apply these techniques in other languages to achieve a similar outcome.

## Capturing interactions

We will need the following data to produce the sequence diagram

1. HTTP request into the system under test generated by the API consumer
1. Database query and result
1. HTTP interactions with external services
1. HTTP response returned by the system under test

### HTTP interactions

We can capture these interactions by defining a behavioural test against the API - in these tests, we treat the API as a black box by defining a HTTP request and an expected response.

```go
func TestGetArtist(t *testing.T) {
  req := httptest.NewRequest("GET", "/artists/123", nil)
  res := httptest.NewRecorder()

  handler.ServeHTTP(res, req)

  assert.Equal(t, http.StatusOK, res.Result().Status)
}
```

The `req` and `res` variables give us the data required in `1` and `4`. We can build a minimal lightweight DSL on top of this test which will abstract away code that records the interactions. We convert the above code to the following

```go
func TestGetArtist(t *testing.T) {
  apitest.New().
    Handler(handler).
    Get("/artists/123").
    Expect(t).
    Status(http.StatusOK).
    End()
}
```

where `End()` runs the test code whilst persisting the interactions into memory so they can be analysed later. `End()` might be implemented as follows

```go
func (r *APITest) End() {
  // run the test
  req := httptest.NewRequest(r.method, r.path, r.body)
  res := httptest.NewRecorder()
  handler.ServeHTTP(res, req)
  
  // capture interactions
  r.request = req
  r.response = res.Result()
  
  // perform assertions
  assert.Equal(t, r.expectedStatus, res.Result().Status)
}
```

### Database interactions

To capture database interactions we use the decorator pattern and implement a custom database driver that wraps the driver used in production code. This allows us to intercept the queries and capture the query string and result. In Go you can register a custom database driver like so

```go
sql.Register("myDriver", myDriver)
```

where `myDriver` is a struct that implements the `sql.Driver` and has a member field with a reference to the real Postgresql driver. When implementing the driver methods we record the SQL query, call the real Postgresql driver then capture the result.

```go
func (d *RecordingDriver) Exec(query string, args []driver.Value) (driver.Result, error) {
  // record the query
  d.apiTest.dbQuery = fmt.Sprintf("%s %+v", query, args)
  
  // perform the query
  res, err := d.driver.Exec(query, args)
  ...
  
  // record the result
  d.apiTest.dbResult = fmt.Sprintf("Affected rows: %d", res.RowsAffected()),
}
```

The nice thing here is that this code will work for all SQL libraries, `mysql`, `postgresql`, `sqlite` etc and also ORMs like `gorm`. This is possible because we are working with low-level SQL code at the driver level. 

This approach can be used to capture interactions for arbitrary data sources such as Amazon S3 and SQS.

### External services interactions

For behavioural tests it is preferable to mock external services to keep the tests fast, repeatable and reproducible. Integrating with the real external API adds unknown factors that often cause tests to break due to reasons outside of our control. This does not replace end-to-end tests and depending on the nature of the project we might tailor the testing strategy.

To capture HTTP requests to external services we will provide mocks for the external services, then capture any requests that cross the mock. We can implement a simple mocking utility by hijacking the default `http.Transport`. A custom `http.Transport` can be injected into an HTTP client as follows.

```go
http.Client{Transport: myTransport}
```

The transport controls low-level client configuration such as TLS, keep alives, proxies and compression. `http.Transport` is an implementation of `http.RoundTripper` which is the interface that executes an HTTP transaction. If we provide an implementation of `http.RoundTripper` we can capture the HTTP request and return a mock response.

```go
type mockTransport struct {
	// inject apitest struct here so we can record mock interactions
}

func (r *mockTransport) RoundTrip(*http.Request) (*http.Response, error) {
  // return mocks here based on request criteria 
  // also capture request and response
}

func TestGetArtistAlbums(t *testing.T) {
  ...
  // this client should be injected into the application under test
  cli := http.Client{Transport: &mockTransport{}}
}
```

The benefit of this approach versus using a separate mocking tool like `Wiremock` is that we don't need to explicitly manage the lifecycle of the process which runs the mock server. It also means the tests run very fast and get close to unit testing times.

## Transforming interactions into a diagram

We now have a collection of events - HTTP and database interactions. We can transform these events into markup using a tool like [PlantUML](https://github.com/steinfletcher/apitest-plantuml) or the [web sequence diagrams](https://www.websequencediagrams.com) DSL - which has a [javascript library](https://bramp.github.io/js-sequence-diagrams/) that allows us to render diagrams as HTML using simple markup.

```text
client->server: GET /message
server->>client: I am good thanks!
```

This produces a sequence diagram.

![hello world sequence diagram!]({{ "/images/simple-sequence-diagram.svg" }})

We can loop over the interactions we captured and generate this markup to produce a sequence diagram and event log. See code [here](https://github.com/steinfletcher/apitest/blob/master/diagram.go#L156) to accomplish this.
 
## Recap

* To generate the sequence diagram 4 pieces of data were necessary to capture
    * Initial HTTP request into the application under test
    * Interactions with external HTTP APIs. We mocked these.
    * Arbitrary datasource interactions, e.g. a database
    * HTTP response to a consumer of the application under test
* We built a high-level DSL on top of Go's `httptest` package to abstract away recording of interactions
* The captured events were transformed into markup using [web sequence diagrams js](https://bramp.github.io/js-sequence-diagrams/). In `apitest` we iterate over the events and generate the DSL. There is also an [extension package](https://github.com/steinfletcher/apitest-plantuml) to render PlantUML markup.
* The code to generate sequence diagrams is now integrated into [apitest](https://apitest.dev).

## Useful links

* [web sequence diagrams js](https://bramp.github.io/js-sequence-diagrams/)
* [apitest behavioural testing library](https://apitest.dev)
* [apitest examples](https://github.com/steinfletcher/apitest/tree/master/examples)
* [PlantUML extension package for apitest](https://github.com/steinfletcher/apitest-plantuml)