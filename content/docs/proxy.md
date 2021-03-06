---
title: proxy
type: docs
directive: true
---

proxy facilitates both a basic reverse proxy and a robust load balancer. The proxy has support for multiple backends and adding custom headers. The load balancing features include multiple policies, health checks, and failovers. Caddy can also proxy WebSocket connections.

This middleware adds a [placeholder](/docs/placeholders) that can be used in [log](/docs/log) formats: **{upstream}** - the name of the upstream host to which the request was proxied.

### Syntax

In its most basic form, a simple reverse proxy uses this syntax:

<code class="block"><span class="hl-directive">proxy</span> <span class="hl-arg"><i>from to</i></span></code>

*   **from** is the base path to match for the request to be proxied
*   **to** is the destination endpoint to proxy to (may include a port range)

However, advanced features including load balancing can be utilized with an expanded syntax:

<code class="block"><span class="hl-directive">proxy</span> <span class="hl-arg"><i>from to...</i></span> {
	<span class="hl-subdirective">policy</span> random | least_conn | round_robin | ip_hash
	<span class="hl-subdirective">fail_timeout</span> <i>duration</i>
	<span class="hl-subdirective">max_fails</span> <i>integer</i>
	<span class="hl-subdirective">max_conns</span> <i>integer</i>
	<span class="hl-subdirective">try_duration</span> <i>duration</i>
	<span class="hl-subdirective">try_interval</span> <i>duration</i>
	<span class="hl-subdirective">health_check</span> <i>path</i>
	<span class="hl-subdirective">health_check_interval</span> <i>interval_duration</i>
	<span class="hl-subdirective">health_check_timeout</span> <i>timeout_duration</i>
	<span class="hl-subdirective">header_upstream</span> <i>name value</i>
	<span class="hl-subdirective">header_downstream</span> <i>name value</i>
	<span class="hl-subdirective">keepalive</span> <i>number</i>
	<span class="hl-subdirective">without</span> <i>prefix</i>
	<span class="hl-subdirective">except</span> <i>ignored_paths...</i>
	<span class="hl-subdirective">upstream</span> <i>to</i>
	<span class="hl-subdirective">insecure_skip_verify</span>
	<span class="hl-subdirective"><i>preset</i></span>
}</code>

*   **from** is the base path to match for the request to be proxied.
*   **to** is the destination endpoint to proxy to. At least one is required, but multiple may be specified. If a scheme (http/https) is not specified, http is used. Unix sockets may also be used by prefixing "unix:".
*   **policy** is the load balancing policy to use; applies only with multiple backends. May be one of random, least_conn, round_robin, or ip_hash. Default is random.
*   **fail\_timeout** specifies how long to remember a failed request to a backend. A timeout > 0 enables request failure counting and is required for load balancing between backends in case of failures. If the number of failed requests accumulates to the max\_fails value, the host will be considered down and no requests will be routed to it until failed requests begin to be forgotten. By default, this is disabled (0s), meaning that failed requests will not be remembered and the backend will always be considered available. Must be a duration value (like "10s" or "1m").
*   **max\_fails** is the number of failed requests within fail\_timeout that are needed before considering a backend to be down. Not used if fail_timeout is 0. Must be at least 1. Default is 1.
*   **max\_conns** is the maximum number of concurrent requests to each backend.  The default value of 0 indicates no limit.  When the limit is reached, additional requests will fail with Bad Gateway (502).
*   **try_duration** is how long to try selecting available upstream hosts for each request. By default, this retry is disabled ("0s"). Clients may hang for this long while the proxy tries to find an available upstream host. This value is only used if a request to the initially-selected upstream host fails.
*   **try\_interval** is how long to wait between selecting another upstream host to handle a request. Default is 250ms. Only relevant when a request to an upstream host fails. Be aware that setting this to 0 with a non-zero try\_duration can result in very tight looping and spin the CPU if all hosts stay down.
*   **health\_check** will use _path_ to check the health of each backend. If a backend returns a status code of 200-399, then that backend is considered healthy. If it doesn't, the backend is marked as unhealthy for at least _health\_check\_interval_ and no requests are routed to it. If this option is not provided then health checks are disabled.
*   **health\_check\_interval** specifies the time between each health check on unhealthy backends. The default interval is 30 seconds ("30s").
*   **health_check_timeout** sets a deadline for health check requests. If a health check does not respond within _health\_check\_timeout_, the health check is considered failed. The default is 60 seconds ("60s").
*   **header\_upstream** sets headers to be passed to the backend. The field name is _name_ and the value is _value_. This option can be specified multiple times for multiple headers, and dynamic values can also be inserted using [request placeholders](/docs/placeholders). By default, existing header fields will be replaced, but you can add/merge field values by prefixing the field name with a plus sign (+). You can remove fields by prefixing the header name with a minus sign (-) and leaving the value blank.
*   **header\_downstream** modifies response headers coming back from the backend. It works the same way header\_upstream does.
*   **keepalive** is the maximum number of idle connections to keep open to the backend. Enabled by default; set to 0 to disable keepalives. Set to a higher value on busy servers that are properly tuned.
*   **without** is a URL prefix to trim before proxying the request upstream. A request to /api/foo without /api, for example, will result in a proxy request to /foo.
*   **except** is a space-separated list of paths to exclude from proxying. Requests that match any of _ignored\_paths_ will be passed thru.
*   **upstream** specifies another backend. It may use a port range like ":8080-8085" if desired. It is often used multiple times when there are many backends to route to.
*   **insecure\_skip\_verify** overrides verification of the backend TLS certificate, essentially disabling security features over HTTPS.
*   **preset** is an optional shorthand way of configuring the proxy to meet certain conditions. See presets below.

<mark class="block">Note that in order to do proper, redundant load balancing in the event of failures, you **must** set fail\_timeout and try\_duration to values &gt; 0.</mark>

Everything after the first _to_ is optional, including the block of properties enclosed by curly braces.

### Presets

The following presets are available:

*   **websocket**  
    Indicates this proxy is forwarding WebSocket connections. It is shorthand for: <code class="block"><span class="hl-subdirective">header_upstream</span> Connection {>Connection}
<span class="hl-subdirective">header_upstream</span> Upgrade {>Upgrade}</code>
	<mark class="block">HTTP/2 does not support protocol upgrade.</mark>

*   **transparent**  
    Passes thru host information from the original request as most backend apps would expect. Shorthand for: <code class="block"><span class="hl-subdirective">header_upstream</span> Host {host}
<span class="hl-subdirective">header_upstream</span> X-Real-IP {remote}
<span class="hl-subdirective">header_upstream</span> X-Forwarded-For {remote}
<span class="hl-subdirective">header_upstream</span> X-Forwarded-Proto {scheme}</code>

### Policies

There are three load balancing policies available:

*   **random** (default) - Randomly select a backend
*   **least_conn** - Select backend with the fewest active connections
*   **round_robin** - Select backend in round-robin fashion

### Examples

Proxy all requests within /api to a backend system:

<code class="block"><span class="hl-directive">proxy</span> <span class="hl-arg">/api localhost:9005</span></code>

Load-balance all requests between three backends (using random policy):

<code class="block"><span class="hl-directive">proxy</span> <span class="hl-arg">/ web1.local:80 web2.local:90 web3.local:100</span></code>

Same as above, but round-robin style:

<code class="block"><span class="hl-directive">proxy</span> <span class="hl-arg">/ web1.local:80 web2.local:90 web3.local:100</span> {
	<span class="hl-subdirective">policy</span> round_robin
}</code>

With health checks and proxy headers to pass hostname, IP, and scheme upstream:

<code class="block"><span class="hl-directive">proxy</span> <span class="hl-arg">/ web1.local:80 web2.local:90 web3.local:100</span> {
	<span class="hl-subdirective">policy</span> round_robin
	<span class="hl-subdirective">health_check</span> /health
	<span class="hl-subdirective">transparent</span>
}</code>

Proxy WebSocket connections:

<code class="block"><span class="hl-directive">proxy</span> <span class="hl-arg">/stream localhost:8080</span> {
	<span class="hl-subdirective">websocket</span>
}</code>

Proxy everything except requests to /static or /robots.txt:

<code class="block"><span class="hl-directive">proxy</span> <span class="hl-arg">/ backend:1234</span> {
	<span class="hl-subdirective">except</span> /static /robots.txt
}</code>
