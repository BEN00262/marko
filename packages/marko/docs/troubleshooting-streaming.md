# Troubleshooting HTTP Streams

[The way Marko streams HTML](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Transfer-Encoding) is old and well-supported, but default configurations and assumptions by other software can foil it. This page describes some known culprits that may buffer your Node server’s output HTTP streams.

## Reverse proxies/load balancers

- Turn off proxy buffering, or if you can’t, set the proxy buffer sizes to be reasonably small.

- Make sure the “upstream” HTTP version is 1.1 or higher; HTTP/1.0 and lower do not support streaming.

- Some software doesn’t support HTTP/2 or higher “upstream” connections at all or very well — if your Node server uses HTTP/2, you may need to downgrade.

- Automatic gzip/brotli compression may have their buffer sizes set too high; you can tune their buffers to be smaller for faster streaming in exchange for slightly worse compression.

- Check if “upstream” connections are `keep-alive`: overhead from closing and reopening connections may delay responses.

### NGiNX

Most of NGiNX’s relevant parameters are inside [its builtin `http_proxy` module](http://nginx.org/en/docs/http/ngx_http_proxy_module.html#proxy_buffering):

```nginx
proxy_http_version 1.1; # 1.0 by default
proxy_buffering off; # on by default
```

### Apache

Apache’s default configuration works fine with streaming, but your host may have it configured differently. The relevant Apache configuration is inside [its `mod_proxy` and `mod_proxy_*` modules](https://httpd.apache.org/docs/2.4/mod/mod_proxy.html) and their [associated environment variables](https://httpd.apache.org/docs/2.4/env.html).

## CDNs

Content Delivery Networks (CDNs) consider efficient streaming one of their best features, but it may be off by default or if certain features are enabled.

- For Fastly or another provider that uses VCL configuration, check [if backend responses have `beresp.do_stream = true` set](https://developer.fastly.com/reference/vcl/variables/backend-response/beresp-do-stream/).

- Some [Akamai features designed to mitigate slow backends can ironically slow down fast chunked responses](https://community.akamai.com/customers/s/question/0D50f00006n975d/enabling-chunked-transfer-encoding-responses). Try toggling off Adaptive Acceleration, Ion, mPulse, Prefetch, and/or similar performance features. Also check for the following in the configuration:

  ```html
  <network:http.buffer-response-v2>off</network:http.buffer-response-v2>
  ```

## Node.js itself

For extreme cases where [Node streams very small HTML chunks with its built-in compression modules](https://github.com/marko-js/marko/pull/1641), you may need to tweak the compressor stream settings. Here’s an example with `createGzip` and its `Z_PARTIAL_FLUSH` flag:

```js
const http = require("http");
const zlib = require("zlib");

const markoTemplate = require("./something.marko");

http
  .createServer(function (request, response) {
    response.writeHead(200, { "content-type": "text/html;charset=utf-8" });
    const templateStream = markoTemplate.stream({});
    const gzipStream = zlib.createGzip({
      flush: zlib.constants.Z_PARTIAL_FLUSH
    });
    templateStream.pipe(outputStream).pipe(response);
  })
  .listen(80);
```