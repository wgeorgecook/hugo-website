+++
title = "Go Http Mocks and Testing"
date = "2024-12-30T20:24:38-05:00"
author = "William George Cook"
tags = ["go", "golang", "testing", "mocking", "http"]
keywords = ["go", "golang", "testing", "mocking", "http"]
description = "A cursory overview of testing HTTP servers and clients."
showFullContent = false
readingTime = false
hideComments = true
+++

# Testing HTTP Endpoints 
Over Christmas I decided to do a small dive into HTTP testing in Go after discovering the [net/http/httptest library](https://pkg.go.dev/net/http/httptest) while writing my [Go Ignite Talk](hhttps://williamcook.dev/posts/why-go-ignite-talk/). 

# The Code
The code discussed here is from [my Github repository](https://github.com/wgeorgecook/http-testing/tree/main). All of the code there is pretty well documented and if you have a good understanding of [Go interfaces](https://williamcook.dev/posts/botanical_exploration_of_abstract_types_in_go/) and patterns (factory and dependency injection specifically) you would do well to read the code rather than this post. 

## Testing the API Server
First, let's examine the HTTP server tests. You can find the handler in `http-testing/internal/pkg/server/handlers.go`. The code is very straightforward.
```
func getResourceHandler(w http.ResponseWriter, req *http.Request) {
	idCheck, ok := req.URL.Query()["id"]
	if !ok || idCheck[0] == "" {
		http.Error(w, errs.ErrIdRequired.Error(), http.StatusBadRequest)
		return
	}

	id := idCheck[0]
	if id == "3" {
		http.Error(w, errs.ErrIdNotFound.Error(), http.StatusNotFound)
		return
	}

	r := resources.Resource{ID: id}
	rBytes, err := json.Marshal(r)
	if err != nil {
		http.Error(w, err.Error(), http.StatusInternalServerError)
	}

	w.Header().Set("Content-Type", "application/json")
	written, err := w.Write(rBytes)
	if err != nil {
		http.Error(w, err.Error(), http.StatusInternalServerError)
		return
	}
	w.Header().Set("Content-Length", strconv.Itoa(written))
	return
}
```

Of note, it 
1. Ensures that a `id` query parameter was passed in on the request.
2. Returns `HTTP NOT FOUND` if the ID provided is specifically 3.
3. Returns the resource with any other ID. 

### HTTP Test
Using the standard library tool `net/http/httptest`, I am able to record a mock request. The action happens in this function found in `http-testing/internal/pkg/server/handlers_test.go`

```
var getResp = func(id string) (*http.Response, []byte) {
    req := httptest.NewRequest("GET", "http://example.com/api/v1/resources?id="+id, nil)
    w := httptest.NewRecorder()
    getResourceHandler(w, req)

    resp := w.Result()
    defer resp.Body.Close()
    body, _ := io.ReadAll(resp.Body)

    return resp, body
}
```

In ~10 lines of code I'm able to call the handler with whatever data I want to pass into it and get an `*http.Response` and `[]byte` back to unmarshal into whatever object I need to. No custom impmentation needed on this side! I'll leave it as an exercise to the reader to check out the actual test cases and see how I am asserting the data my handler returns is correct. 

## Testing the API Consumer
The API consumer is responsible for actually making network requests, so testing it's behavior is a little more complicated. The first step to success is using [dependency injection](https://www.freecodecamp.org/news/a-quick-intro-to-dependency-injection-what-it-is-and-when-to-use-it-7578c84fa88f/) via a provided HTTP client. So let's take a look at the `api.Client`. 

```
// Client is the default way to interact with
// the API SDK. Requests to the remote endpoints
// are made via the HttpClient provided.
type Client struct {
	HttpClient HttpClient
}

```

`api.Client` requires an `HttpClient`, which we descibe as an interface with a `Get` method. 
```
// HttpClient is an interface that partially
// supports the http.Client interface and
// allows mocking requests made on it.
type HttpClient interface {
	Get(url string) (resp *http.Response, err error)
}
```

Since the [http.Client](https://pkg.go.dev/net/http#Client.Get) satisfies this, we can always pass in that when constructing a new Client. But it leaves the option to allow the consumer to pass in something more custom. This is exactly how we are able to test this. 

### Mocking Requests
A custom struct that satisfies the `HttpClient` interface is tucked away in `http-testing/internal/pkg/utils/mocks/httpClientMock.go`. 
```
// HTTPClientMock is the type that receives requests
// during the tests for an api.Client.
type HTTPClientMock struct{}

// Get implements the api.HttpClient interface and
// receives requests made by the api.Client when
// client.HttpClient.Get is called.
// Note: Specific behaviors expected from the
// remote endpoint MUST be updated or implemented
// in this function for the tests to remain valid.
func (c *HTTPClientMock) Get(endpoint string) (*http.Response, error) {
	fullUrl, err := url.Parse(endpoint)
	if err != nil {
		return nil, err
	}
	id := fullUrl.Query().Get("id")
	switch id {
	case "":
		return &http.Response{
			StatusCode: http.StatusBadRequest,
			Body:       io.NopCloser(strings.NewReader(errs.ErrIdRequired.Error())),
		}, errs.ErrIdRequired
	case "1":
		return &http.Response{
			StatusCode: http.StatusOK,
			Body:       io.NopCloser(strings.NewReader(MockResourseId1)),
		}, nil
	case "3":
		return &http.Response{
			StatusCode: http.StatusNotFound,
			Body:       io.NopCloser(strings.NewReader(errs.ErrIdNotFound.Error())),
		}, errs.ErrIdNotFound
	}

	return &http.Response{StatusCode: http.StatusNotImplemented}, errs.ErrUnimplemented
}
```

When testing the client, all of the expected behavior must be implemented in this `HTTPClientMock.Get()` function. Notice here that things are exactly as you would expect if you read the tests from the API server. An `id` query param is required, but any time `id=3` is provided you'll get an `HTTP NOT FOUND` status code back. Try and understand how this client hijacks the requests made in the `client.GetResourceByID()` request via dependency injection as I mentioned above. You can find the tests in `http-testing/internal/pkg/api/http_test.go`. 

# In Closing
This is a _very brief, very contrived_ way to get some HTTP tests done for your API servers and your consumers. I encourage you to use this as a baseline and try and implement mock tests in your own code. Experiment and really understand how things work under the hood and I'm sure your own code quality will get better as a side effect! 