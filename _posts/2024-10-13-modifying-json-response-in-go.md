---
layout: post
title: "Modifying `NewSingleHostReverseProxy` Response Data in Go without HTTP Errors"
date: 2024-10-13 04:00:00 +0800
categories: code
tags: golang go proxy json http gin
---

## Background

Setting up an HTTP proxy in Go is straightforward with the built-in [NewSingleHostReverseProxy](https://pkg.go.dev/net/http/httputil#NewSingleHostReverseProxy). When used with [Gin](https://github.com/gin-gonic/gin) as middleware, the code looks like this:

```go
router := gin.New()
proxy := httputil.NewSingleHostReverseProxy(lcdURL)
proxyHandler := func(c *gin.Context) {
  proxy.ServeHTTP(c.Writer, c.Request)
}
router.Use(proxyHandler)
```

But what if we want to modify the proxied response before sending it back to the client? For example, we might want to hide PII before serving internal data to third parties. For simplicity, let's assume the body is in JSON format.

## Built-in Functions vs. Middleware

Sadly, `NewSingleHostReverseProxy` doesn't directly support response modification. However, the [ReverseProxy](https://pkg.go.dev/net/http/httputil#ReverseProxy) instance it returns allows us to use the method `ModifyResponse`, which allows us to define a function to modify the response:

```go
proxy.ModifyResponse = func(r *http.Response) error {
  b, err := io.ReadAll(r.Body)
  if err != nil {
    return err
  }
  defer r.Body.Close()

  var jsonObject map[string]interface{}
  err = json.Unmarshal(b, &jsonObject)
  if err != nil {
    return err
  }

  // Modify jsonObject here

  newBody, err := json.Marshal(jsonObject)
  if err != nil {
    return err
  }

  r.Body = io.NopCloser(bytes.NewReader(newBody))
  return nil
}
```

However, if you're running the proxy alongside other API routes, you might want a unified way to modify all responses. Writing the rewrite logic as Gin middleware can achieve this.

```go
func filterJsonBody() gin.HandlerFunc {
  return func(c *gin.Context) {
    // Create our own writer
    wb := &copyWriter{
      body:           &bytes.Buffer{},
      ResponseWriter: c.Writer,
    }

    // Inject it into gin context
    c.Writer = wb

    // Call the next handler
    c.Next()

    // Handle response modification at the end of handler chain
    originBodyBytes := wb.body.Bytes()

    var jsonObject map[string]interface{}
    err := json.Unmarshal(originBodyBytes, &jsonObject)
    if err != nil {
      c.AbortWithError(http.StatusInternalServerError, err)
      return
    }

    // Modify jsonObject here

    newBody, err := json.Marshal(jsonObject)
    if err != nil {
      c.AbortWithError(http.StatusInternalServerError, err)
      return
    }

    wb.ResponseWriter.Write(newBody)
  }
}

type copyWriter struct {
  gin.ResponseWriter
  body *bytes.Buffer
}

func (cw *copyWriter) Write(b []byte) (int, error) {
  return cw.body.Write(b)
}
```

Add it at the beginning of the router handler chain, so that our `copyWriter` instance would replace the `Writer` in the Gin context for all routes.

```go
router := gin.New()
router.Use(filterJsonBody())

// Other routes
router.Use(proxyHandler)
```

## HTTP Error?

If you add the middleware as shown and then use curl to test the API, you may encounter errors like:

- `HTTP/2 stream 1 was not closed cleanly: INTERNAL_ERROR (err 2)`
- `curl: (18) transfer closed with x bytes remaining to read`

Which of these would show up depends on your infrastructure setup. For example, cloud hosting load balancers would likely be in HTTP/2. The HTTP/2 error can be cryptic, but the curl error hints at a `Content-Length` header mismatch. One way to confirm the issue is to make curl show the HTTP headers by using `curl -v`. Indeed, the `Content-Length` header is set as the original body length, instead of the modified one.

### Fixing the Content-Length Header in `ModifyResponse` Approach

Fixing this issue in `ModifyResponse` is straightforward; we just need to set the `Content-Length` header to the new body length:

```go
r.ContentLength = int64(len(newBody))
r.Header.Set("Content-Length", strconv.Itoa(len(newBody)))
```

### Fixing the Content-Length Header in Gin Middleware Approach

In the middleware approach, ideally, we would also rewrite the `Content-Length` header with the new correct length. However, in the method we would override here, `WriteHeader`, we can't directly access the `http.Response` object. Instead, a workaround is to use `Transfer-Encoding: chunked` to signal that the response body is sent in chunks. This way, we don't need to set the `Content-Length` header at all:

```go
func (cw *copyWriter) WriteHeader(statusCode int) {
  cw.Header().Del("Content-Length")
  cw.Header().Set("Transfer-Encoding", "chunked")
  cw.ResponseWriter.WriteHeader(statusCode)
}
```

One downside of using `chunked` encoding is that some HTTP clients and proxies may not cache them properly. Since I use Nginx in my setup, which caches small chunked responses, this tradeoff is acceptable for my use case.

### Conclusion

Setting up a reverse proxy in Go is easy; the ability to modify the response allows even more flexibility in use cases. By using Gin middleware, we easily apply a unified modification logic to all routes. However, be aware of the `Content-Length` header issue when modifying the response body, and choose the appropriate solution wisely based on your setup.
