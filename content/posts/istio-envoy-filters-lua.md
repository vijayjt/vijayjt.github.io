+++ 
draft = false
date = 2024-01-09T11:05:23Z
title = "Istio Envoy Filters with Lua"
description = ""
slug = ""
authors = []
tags = ["Kubernetes", "Istio", "Envoy", "Lua"]
categories = []
externalLink = ""
series = []
+++

Istio [Envoy Filters](https://istio.io/latest/docs/reference/config/networking/envoy-filter/) provides a way customise Envoy's behaviour. They can be useful when you have a requirement that cannot be fulfilled out of the box by Istio.

> **Warning**
>
> When using Envoy Filters we are using low-level APIs that can change between Istio versions. It is therefore important that you have robust tests in place
> and review change logs/release notes carefully for changes that may break your filters.

Envoy itself has a number of [built-in](https://www.envoyproxy.io/docs/envoy/latest/configuration/http/http_filters/http_filters) filters but we are specifically going to talk about [Lua](https://www.envoyproxy.io/docs/envoy/latest/configuration/http/http_filters/lua_filter) filters which allows Lua code to be executed during both the request and response flows.

It is worth noting that Lua filters have some drawbacks:

* You're effectively embedding Lua code inside YAML which is not a great developer experience. Even syntax errors may not be caught until you try to deploy the filter.
* It's hard to test the Lua code embedded in the YAML.

You could use the ` sidecar.istio.io/userVolume` and `sidecar.istio.io/userVolumeMount` [annotations](https://istio.io/latest/docs/reference/config/annotations/) to mount Lua code that is in a ConfigMap as a volume in the istio-proxy containers. This could for example allow you to attempt to modularise the code and create libraries etc. but this is a pain to maintain.

An alternative is to use [WasmPlugins](https://istio.io/latest/docs/reference/config/proxy_extensions/wasm-plugin/) and write your code in Go, Rust or other supported languages. We will discuss this option in another post.

## Full example

The full example is shown below - this filter is used to modify the format of error responses (e.g. HTTP Status code 400 and above), we will discuss the various elements of this filter.

```YAML
apiVersion: networking.istio.io/v1alpha3
kind: EnvoyFilter
metadata:
  name: my-custom-filter
  namespace: istio-system # Namespace where istio gateway pods are actually running
spec:
  workloadSelector:
    labels:
      app: istio-ingressgateway
  configPatches:
  # Patch that creates "global" lua filter that does nothing useful
  - applyTo: HTTP_FILTER
    match:
      listener:
        filterChain:
          filter:
            name: envoy.filters.network.http_connection_manager
            subFilter:
              name: envoy.filters.http.router
    patch:
      operation: INSERT_BEFORE
      value:
        name: envoy.lua
        typed_config:
          '@type': type.googleapis.com/envoy.extensions.filters.http.lua.v3.Lua
          inlineCode: |
            function envoy_on_request(request_handle)
              -- Empty lua function
            end
  # Filter for http route that overrides "global" filter lua source code
  - applyTo: HTTP_ROUTE
    match:
      context: GATEWAY
      routeConfiguration:
        vhost:
          route:
          name: "foo-virtual-service.bar-namespace.svc.cluster.local:443"
    patch:
      operation: MERGE
      value:
        metadata:
          filter_metadata:
            envoy.filters.http.lua:
              default_trace_id: "00-0aa0000000aa00aa0000aa000a00000a-a0aa0a0000000000-00"
              default_problem_title: "Istio API Ingress gateway returned an error"
              default4xx_problem_type: "https://datatracker.ietf.org/doc/html/rfc9110#name-client-error-4xx"
              default5xx_problem_type: "https://datatracker.ietf.org/doc/html/rfc9110#name-client-error-5xx"
              problem_type_uri_map:
                - 400: "https://tools.ietf.org/html/rfc9110#section-15.5.1"
                - 401: "https://tools.ietf.org/html/rfc9110#section-15.5.2"
                - 403: "https://tools.ietf.org/html/rfc9110#section-15.5.4"
                - 404: "https://tools.ietf.org/html/rfc9110#section-15.5.5"
                - 405: "https://tools.ietf.org/html/rfc9110#section-15.5.6"
                - 406: "https://tools.ietf.org/html/rfc9110#section-15.5.7"
                - 408: "https://tools.ietf.org/html/rfc9110#section-15.5.9"
                - 409: "https://tools.ietf.org/html/rfc9110#section-15.5.10"
                - 412: "https://tools.ietf.org/html/rfc9110#section-15.5.13"
                - 415: "https://tools.ietf.org/html/rfc9110#section-15.5.16"
                - 422: "https://tools.ietf.org/html/rfc4918#section-11.2"
                - 426: "https://tools.ietf.org/html/rfc9110#section-15.5.22"
                - 500: "https://tools.ietf.org/html/rfc9110#section-15.6.1"
                - 502: "https://tools.ietf.org/html/rfc9110#section-15.6.3"
                - 503: "https://tools.ietf.org/html/rfc9110#section-15.6.4"
                - 504: "https://tools.ietf.org/html/rfc9110#section-15.6.5"
        name: envoy.filters.http.lua
        typed_per_filter_config:
          envoy.filters.http.lua:
            '@type': type.googleapis.com/envoy.extensions.filters.http.lua.v3.LuaPerRoute
            source_code:
              inline_string: |
                -- Extract request info and set as dynamic metadata so we can use it in the response
                function envoy_on_request(request_handle)
                  headers = request_handle:headers()
                  request_path = headers:get(":path")

                  for key, value in pairs (headers) do
                      if key == nil or value == nil then
                      request_handle:logInfo("[CUSTOM FILTER] NIL")
                      else
                      request_handle:logInfo("[CUSTOM FILTER] Header:" .. key .. " -> " .. value)
                      end
                  end

                  -- If the W3C traceparent header is not present use the istio x-request-id instead
                  -- are there any cases where even this may be nil?
                  trace_id = headers:get("traceparent")
                  if trace_id == nil then
                    trace_id = headers:get("x-request-id")
                  end

                  if trace_id == nil then
                    local metadata = response_handle:metadata()
                    trace_id = metadata:get("default_trace_id")
                  end

                  request_handle:streamInfo():dynamicMetadata():set("context", "xx.request.traceid", trace_id)
                  request_handle:streamInfo():dynamicMetadata():set("context", "xx.request.path", request_path)
                end

                -- Returns the problem type URI for a given HTTP status code 
                function get_problem_type_uri(status_code, response_handle)
                  -- get filter metadata - which is different to dynamic metadata
                  local metadata = response_handle:metadata()
                  problem_type_uri_map = metadata:get("problem_type_uri_map")
                  default4xx_problem_type = metadata:get("default4xx_problem_type")
                  default5xx_problem_type = metadata:get("default5xx_problem_type")
                  problem_type_uri = ""
                  for id, uri_lookup in pairs(problem_type_uri_map) do
                    for code, uri in pairs(uri_lookup) do
                      if code == status_code then
                        problem_type_uri = uri
                      end
                    end
                  end

                  if problem_type_uri == "" and status_code:match('^4(.*)')  then 
                    problem_type_uri = default4xx_problem_type
                  elseif problem_type_uri == "" and status_code:match('^5(.*)')  then 
                    problem_type_uri = default5xx_problem_type
                  end

                  return problem_type_uri

                end

                -- Customise the response
                function envoy_on_response(response_handle)
                  headers = response_handle:headers()
                  original_content_type = headers:get("content-type")
                  response_content_type = "application/problem+json"
                  response_handle:logInfo(string.format("ORIGINAL CONTENT TYPE: %s", original_content_type))
                  resp_status_str = headers:get(":status")
                  resp_status_int = tonumber(resp_status_str)

                  request_path = response_handle:streamInfo():dynamicMetadata():get("context")["xx.request.path"]
                  trace_id = response_handle:streamInfo():dynamicMetadata():get("context")["xx.request.traceid"]

                  problem_type_uri = get_problem_type_uri(resp_status_str, response_handle)

                  -- note we can't use this for 404 errors apparently so a separate filter is required for that
                  if resp_status_int >= 400 and resp_status_int <= 599 and original_content_type ~= "text/html" then
                   response_handle:logInfo(string.format("http status code is %s, customising error response", resp_status_str))
                   response_handle:headers():replace("content-type", response_content_type)

                   local body_str = ""
                   local body = response_handle:body()
                   if body == nil then
                     body_str = "Service mesh returned http " .. resp_status_str .. " status code."
                   else
                     body_str = tostring(body:getBytes(0, body:length()))
                   end

                   -- The double brackets are a string concatenation / heredocs like operator
                   -- We un-indent to avoid extraneous leading spaces in the response JSON
                   local resp=[[
                {
                  "type": "]] .. problem_type_uri .. [[",
                  "title": "service mesh returned an HTTP ]] .. resp_status_str .. [[ error.",
                  "status": ]] .. resp_status_str .. [[,
                  "instance": "]] .. request_path .. [[",
                  "trace_id": "]] .. trace_id .. [[",
                  "detail": "]] .. body_str .. [["
                }
                ]]

                   -- Pass in always_wrap_body to ensure Istio/Envoy always returns a body object even if the body is empty.
                   -- Certain types of errors returned by Istio will have an empty body and without always_wrap_body this would result an exception being thrown in our code.
                   always_wrap_body = true
                   response_handle:body(always_wrap_body):setBytes(resp)

                  -- end if http status
                  end
                end
```

## Scoping the filter

In this case we create the filter in the istio-system namespace and it specifically applies to an ingress [Gateway](https://istio.io/latest/docs/reference/config/networking/gateway/) with the label `istio-ingressgateway` this allows you to target the filter to specific gateways.

```YAML
apiVersion: networking.istio.io/v1alpha3
kind: EnvoyFilter
metadata:
  name: my-custom-filter
  namespace: istio-system # Namespace where istio gateway pods are actually running
spec:
  workloadSelector:
    labels:
      app: istio-ingressgateway
```

In this case we have a requirement to only apply the filter for specific routes so:

* We have an empty filter that does nothing
* Then create another filter that is scoped to a specific virtual host `foo-virtual-service.bar-namespace.svc.cluster.local` this should match the cluster local FQDN for your virtual service.

An alternative would be to just check the request URL and if it does not match the one we are interested in do nothing (you can derive the request URL from the `authority` and `path` headers.

```YAML
  configPatches:
  # Patch that creates "global" lua filter that does nothing useful
  - applyTo: HTTP_FILTER
    match:
      listener:
        filterChain:
          filter:
            name: envoy.filters.network.http_connection_manager
            subFilter:
              name: envoy.filters.http.router
    patch:
      operation: INSERT_BEFORE
      value:
        name: envoy.lua
        typed_config:
          '@type': type.googleapis.com/envoy.extensions.filters.http.lua.v3.Lua
          inlineCode: |
            function envoy_on_request(request_handle)
              -- Empty lua function
            end
  # Filter for http route that overrides "global" filter lua source code
  - applyTo: HTTP_ROUTE
    match:
      context: GATEWAY
      routeConfiguration:
        vhost:
          route:
          name: "foo-virtual-service.bar-namespace.svc.cluster.local:443"
```


## Logging and comments

We can add logging using `handle_name:logInfo` to write a info level log message. There are different log levels such as warning, error, info etc. 

```
    -- Example for string formatting
    response_handle:logInfo(string.format("http status code is %s, customising error response", resp_status_str))
    response_handle:logWarn("This is a warning message")
    response_handle:logError("This is an error message")
    response_handle:logCritical("This is a critical message")
```

Comments are prefixed with `--` double dashes.

Note that to display info level logs you would need to change the default logging configuration. This can be achieved with the istioctl command line tool:

`istioctl proxy-config log istio-ingressgateway-5576bb6d65-qzj9j  --level lua:info`

Obviously this is a one-off command and if the container are recreated or the deployment is scaled the logging level will not apply. There is an (at least at the time of writing) alpha feature that allows us to set the component logging level using the annotation `sidecar.istio.io/componentLogLevel`. Refer to the Istio [documentation](https://istio.io/latest/docs/reference/config/annotations/) for further details. The only problem is you need to specify this annotation in the pod template for the ingress gateway containers in the deployment.


## Filter metadata

Filter metadata allows you to define configuration outside of the Lua code that can then be used to modify the behaviour of the Lua code. You may want to do this if you wish to make the code configurable without having to modify the code.

In the example below `default_trace_id` is a metadata item that is a string and `problem_type_uri_map` is a map.

```YAML
      value:
        metadata:
          filter_metadata:
            envoy.filters.http.lua:
              default_trace_id: "00-0aa0000000aa00aa0000aa000a00000a-a0aa0a0000000000-00"
              problem_type_uri_map:
                - 400: "https://tools.ietf.org/html/rfc9110#section-15.5.1"
                - 401: "https://tools.ietf.org/html/rfc9110#section-15.5.2"
                - 403: "https://tools.ietf.org/html/rfc9110#section-15.5.4"
```

We can then access these metadata items from our code like so:

```lua
    local metadata = response_handle:metadata()
    problem_type_uri_map = metadata:get("problem_type_uri_map")
    default4xx_problem_type = metadata:get("default4xx_problem_type")
    default5xx_problem_type = metadata:get("default5xx_problem_type")
    problem_type_uri = ""
    for id, uri_lookup in pairs(problem_type_uri_map) do
      for code, uri in pairs(uri_lookup) do
        if code == status_code then
          problem_type_uri = uri
        end
      end
    end
```

Notice that for maps we can loop through them using a [for loop](https://opensource.com/article/22/11/iterate-over-tables-lua) and the pairs standard library function.

## Dynamic metadata

[Dynamic metadata](https://www.envoyproxy.io/docs/envoy/latest/configuration/advanced/well_known_dynamic_metadata) allows you to insert custom metadata into the request/response chain that your code can then utilise.

We can set dynamic metadata like so:

```lua
    request_handle:streamInfo():dynamicMetadata():set("context", "xx.request.traceid", trace_id)
    request_handle:streamInfo():dynamicMetadata():set("context", "xx.request.path", request_path)
```

We can then retrieve those values like so:

```lua
    request_path = response_handle:streamInfo():dynamicMetadata():get("context")["xx.request.path"]
    trace_id = response_handle:streamInfo():dynamicMetadata():get("context")["xx.request.traceid"]
```

## Calling external services

Your Lua filter can also call external services (they could reside outside of your Kubernetes cluster running your filter or be in another namespace in the cluster).

Here is an example taken directly from the Envoy [documentation](https://www.envoyproxy.io/docs/envoy/latest/configuration/http/http_filters/lua_filter#script-examples):

```lua
function envoy_on_request(request_handle)
  -- Make an HTTP call to an upstream host with the following headers, body, and timeout.
  local headers, body = request_handle:httpCall(
  "lua_cluster",
  {
    [":method"] = "POST",
    [":path"] = "/",
    [":authority"] = "lua_cluster"
  },
  "hello world",
  5000)

  -- Add information from the HTTP call into the headers that are about to be sent to the next
  -- filter in the filter chain.
  request_handle:headers():add("upstream_foo", headers["foo"])
  request_handle:headers():add("upstream_body_size", #body)
end
```

A real world use case may be that you need to implement some custom authentication or authorization that requires calling an external service.
