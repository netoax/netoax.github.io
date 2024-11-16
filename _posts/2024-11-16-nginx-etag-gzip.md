---
layout: post
title: "NGINX, ETag and gzip compression"
author: "João Neto"
categories: networking
tags: [nginx, proxy, http, etag, gzip, browser, networking]
---

| **''Why NGINX doesn't respect the ETag header defined on my server?''**

One of the main optimizations for reaching high performance on web applications is to avoid new requests for resources that were already fetched and their content has not changed. The server can leverage from HTTP headers such as `ETag`, `Last-Modified` headers to indicate the resource payload should not be resend to the clients. Let's explore the `ETag` header particularly.

Based on the [RFC 7232](https://datatracker.ietf.org/doc/html/rfc7232):

> The "ETag" header field in a response provides the current entity-tag for the selected representation, as determined at the conclusion of handling the request.

In other words, `ETag` is an identifier that determines the current version of a resource. We can use this header to know if the requested resource is up-to-date with the client's version, allowing us to cache the resource and preventing the client from receiving the whole content one more time without need.

Another interesting use case is using `ETag` to eliminate collisions when simultaneous clients are trying to modify the same resource, which means it's not restricted to `GET` requests but `PUT` as well.

The `ETag` can be generated with a hash function considering the resource content itself or the resource version stored on the server database. It should be regenerated always the content changes.

There are two categories of `ETag`:

- **Strong**: we can use the identifier to do a reliable byte-to-byte comparison of two entities.
  - Example: `"ETag: <hash-value>"` 
- **Weak**: we cannot use it to do a byte-to-byte comparison even though the resources are semantically identical. The header value is prefixed with a `W/`.
  - Example: `W/"ETag: <hash-value>"` 


### NGINX

NGINX is an HTTP web server widely used for load balancing, content caching, and reverse proxing. The version 1.3.3 supports a directive [etag](https://nginx.org/en/docs/http/ngx_http_core_module.html#etag) on the `ngx_http_core_module`, which means NGINX will automatically generate an ETag header for static contents. Meanwhile, if you are not serving static requests, you can set the `ETag` header on the upstream server as expected.

Remember, we want to answer the question defined in the beginning of the post. For that reason, we will configure a NGINX server to proxy request to an upstream backend server.

Let's consider our backend server here is a product catalog that is serving information from many store products. It exposes a route to fetch the product content.

The NGINX configuration would be quite simple:


```conf
http {
  server {
    listen 8000;

    location /products {
      proxy_pass http://catalog:5001;
    }
  }
}
```

This config means the server will listen on port `8000` for incoming requests then forward these requests to the `catalog` application when matching `/products` path.

The backend server was configured to return an `ETag` header along with the content on the response to `GET /products/123`:

```
➜  catalog git:(initial) ✗ curl -v localhost:8000/products/123
...
< HTTP/1.1 200 OK
< Server: nginx/1.27.2
< Date: Fri, 15 Nov 2024 21:54:57 GMT
< Content-Type: application/json
< Content-Length: 181
< Connection: keep-alive
< Etag: "f8e9eb60772b905355d717bb7ee41da63ed745bd70bcd53764e855773319b893"
<
{
  "name":"Barcelona T-Shirt",
  "sku":123,
  "description":"Barcelona 2024/2025 T-Shirt"
} 
```

Now we have a NGINX proxy forwarding requests to our catalog backend that returns an `ETag` header. The backend was also configured for checking this header on every incoming request from the client. When it detects its presence, it compares the value with the requested resource's `ETag`. If they're equal, the backend returns a `304 - Not Modified` status code without resending the payload again to the client. For the sake of information, the convention is to pass this header via [If-None-Match](https://http.dev/if-none-match) header.

```
➜  catalog git:(initial) ✗ curl -v localhost:8000/products/123 -H "If-None-Match: f8e9eb60772b905355d717bb7ee41da63ed745bd70bcd53764e855773319b893"
...
> If-None-Match: f8e9eb60772b905355d717bb7ee41da63ed745bd70bcd53764e855773319b893
>
< HTTP/1.1 304 Not Modified
< Server: nginx/1.27.2
< Date: Fri, 15 Nov 2024 22:02:49 GMT
< Connection: keep-alive
< X-Content-Type-Options: nosniff
<
``` 

#### Gzip compression

The above use case is simple but imagine the backend returns a huge payload to the client every time a new resource is requested. It's quite common to use a compression method to reduze the payload size and improve the overall performance when fetching these resources.

The NGINX server provides a native module to enable gzip compression when proxying requests to an upstream backend.

We can edit our NGINX configuration file to enable it:

```conf
http {
  server {
    listen 8000;

    gzip on;
    gzip_types application/json;
    gzip_min_length 100;
    gzip_proxied any;

    location /products {
      proxy_pass http://catalog:5001;
    }
  }
}
```

where:

- `gzip_on` enables the module.
- `gzip_types` defines for which content types the compression will be enabled.
- `gzip_min_length` defines the minimum content length for which the compression will be enabled.
- `gzip_proxied` defines the compression will be applied to every proxied request.

Following the same CUrl examples so far, we must now indicate the client can understand `gzip` compression algorithm with the [Accept-Encoding](https://http.dev/accept-encoding) header.

```
catalog git:(initial) ✗ curl -v localhost:8000/products/123 -H "Accept-Encoding: gzip"
...
> Accept-Encoding: gzip
>
< HTTP/1.1 200 OK
< Server: nginx/1.27.2
< Date: Fri, 15 Nov 2024 22:17:52 GMT
< Content-Type: application/json
< Transfer-Encoding: chunked
< Connection: keep-alive
< Cache-Control: max-age=60
< Etag: W/"f8e9eb60772b905355d717bb7ee41da63ed745bd70bcd53764e855773319b893"
< Content-Encoding: gzip
```

As you can see above, the response contains a weak `ETag` rather than a strong one. The definiton of weak validation remains the same. We have an response that's compressed in the `gzip` encoding type and couldn't not be used for byte-to-byte comparison. However, the content is still semantically the same and the server can proceed with a weak validation.

Getting back to the specification:

> An origin server SHOULD change a weak entity-tag whenever it considers prior representations to be unacceptable as a substitute for the current representation.

That's exactly what NGINX server does here. It knows the upstream response is being compressed and the proxied response to the client doesn't have the same byte-to-byte representation. The `ETag` is then automatically converted into a weak one. You can take a look at the NGINX code [here](https://github.com/nginx/nginx/blob/master/src/http/modules/ngx_http_gzip_filter_module.c#L293-L294).