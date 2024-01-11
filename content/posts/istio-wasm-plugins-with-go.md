+++ 
draft = false
date = 2024-01-09T18:23:48Z
title = "Using Istio WASMPlugin to modify Envoy functionality"
description = ""
slug = ""
authors = []
tags = ["Kubernetes", "Istio", "Envoy", "Lua", "WASM", "golang"]
categories = []
externalLink = ""
series = []
+++

In a previous [post]({{< ref "/posts/istio-envoy-filters-lua" >}}), we went through an example of using Envoy Filters to modify the envoy proxy behaviour in Istio.
In this post we will discuss using [WASMPlugin](https://istio.io/latest/docs/reference/config/proxy_extensions/wasm-plugin/).

The WebAssembly for Proxies ABI [Specification](https://github.com/proxy-wasm/spec) provides an application binary interface specification for proxies - while originally developed for Envoy it is not envoy specific. There are SDKs for C++, Go, Rust and AssemblyScript itself.

We will be using the [Go SDK for Proxy-Wasm](https://github.com/tetratelabs/proxy-wasm-go-sdk?tab=readme-ov-file), which uses [TinyGo](https://tinygo.org/) - mainly because I don't know Rust and I have not written C++ for a long time :smile:

One thing to keep in mind is that not all Go [language features](https://tinygo.org/docs/reference/lang-support/) and standard library packages are [supported](https://tinygo.org/docs/reference/lang-support/stdlib/) (in some cases the packages can be imported but not all the tests have passed so your mileage my vary if you use the package anyway).

First install [TinyGo](https://tinygo.org/getting-started/install/linux/) (I assume that you already have Go installed):

```bash
wget https://github.com/tinygo-org/tinygo/releases/download/v0.30.0/tinygo_0.30.0_amd64.deb
sudo dpkg -i tinygo_0.30.0_amd64.deb
```

Then in your directory that will contain your code run `go mod init example-wasm-plugin` (or whatever you want to name your package)

The full example code is available in this [repo](https://github.com/vijayjt/example-istio-wasm-plugin).

I used the [examples](https://github.com/tetratelabs/proxy-wasm-go-sdk/tree/main/examples) provider by Tetrate to get started with structuring my code. The [overview](https://github.com/tetratelabs/proxy-wasm-go-sdk/blob/main/doc/OVERVIEW.md) was also useful to understand the interfaces I needed to satisfy in the code and how these map to envoy configuration.

## Plugin Configuration

Similar to how with the Envoy Filter in Lua we read configuration from metadata, the [WASMPlugin](https://istio.io/latest/docs/reference/config/proxy_extensions/wasm-plugin/) resource allows configuration to be passed via the `PluginConfig` field.

To support this we create two structs: 

```go
// pluginContext implements types.PluginContext interface of proxy-wasm-go SDK.
type pluginContext struct {
        // Embed the default plugin context here,
        // so that we don't need to reimplement all the methods.
        types.DefaultPluginContext
        configuration pluginConfiguration
}

// pluginConfiguration is a type to represent an example configuration for this wasm plugin.
type pluginConfiguration struct {
        targetURLPrefixes []string
        startStatusCode   int
        endStatusCode     int
        problemTypeURIMap map[string]string
        problemTitle string
}
```

The `pluginConfiguration` struct represents the configuration items specific to our plugins We then specify this in the `pluginContext` struct.

The `OnPluginStart` function is called when the plugin is first loaded, this is where we will fetch the plugin configuration and parse it in our `parsePluginConfiguration` function.
 
```go
// Override types.DefaultPluginContext.
func (ctx *pluginContext) OnPluginStart(pluginConfigurationSize int) types.OnPluginStartStatus {
        data, err := proxywasm.GetPluginConfiguration()
        if err != nil && err != types.ErrorStatusNotFound {
                proxywasm.LogCriticalf("error reading plugin configuration: %v", err)
                return types.OnPluginStartStatusFailed
        }
        config, err := parsePluginConfiguration(data)
        if err != nil {
                proxywasm.LogCriticalf("error parsing plugin configuration: %v", err)
                return types.OnPluginStartStatusFailed
        }
        ctx.configuration = config
        return types.OnPluginStartStatusOK
}
```

You can refer to the code in GitHub for the full details of the `parsePluginConfiguration` function but it will simply read the configuration and create an instance of the `pluginConfiguration` struct and return this.

## Modify Filter Chains

The API provides different functions which you must implement to modify request/response headers or body (see [here](https://github.com/tetratelabs/proxy-wasm-go-sdk/blob/b089ccb942195f3ac34dca36b629be91b9c962ff/proxywasm/types/context.go#L210)).

We will only look at two here. `OnHttpRequestheaders` is called when request headers arrive. An example is shown below of how to fetch headers.

```go
func (ctx *customErrorsContext) OnHttpRequestHeaders(numHeaders int, endOfStream bool) types.Action {
   <... removed for brevity ...>
   scheme, err := proxywasm.GetHttpRequestHeader(":scheme")
   if err != nil {
           proxywasm.LogErrorf("failed to get request header scheme. Error: %v", err)
   }

   <... removed for brevity ...>

   return types.ActionContinue
}
```

There are only two Action types, which mean exactly what you would think, ActionContinue means that the host continues the processing and ActionPause will pause processing.

If you are not familiar with http2 then you may be wondering about these headers with ":" in the name, these are new [psuedo-headers](https://datatracker.ietf.org/doc/html/rfc9113#name-request-pseudo-header-field) added for http2, they have a colon in the name so that older clients that do not understand http2 will ignore those headers

A snippet from the `OnHttpResponseBody` function is shown below:

```go
func (ctx *customErrorsContext) OnHttpResponseBody(bodySize int, endOfStream bool) types.Action {
   if !ctx.modifyResponse {
           return types.ActionContinue
   }
   proxywasm.LogInfof("BEGIN OnHttpResponseBody")
   ctx.totalResponseBodySize += bodySize
   if !endOfStream {
           // Wait until we see the entire body before modifying it.
           return types.ActionPause
   }

   originalBody, err := proxywasm.GetHttpResponseBody(0, ctx.totalResponseBodySize)
   if err != nil {
           proxywasm.LogErrorf("failed to get response body. Error: %v", err)
           return types.ActionContinue
   }

   <... removed for brevity ...>
```

The OnHttpResponseBody function will get called multiple times as the response body is processed, when the full response body is received `endOfStream` will be `true`.
Since we intend to read and modify the response body, we also keep track of the response body size so we can use int in the call to `GetHttpResponseBody`.

## Building the code

We can build a wasm binary from our code using tinygo ass follows:

`tinygo build -o ./custom-errors.wasm -scheduler=none -target=wasi ./main.go`

## Tests

The TetrateLabs repo for the Go SDK provides some very good example plugins that include tests so I won't go into a lot of detail about `main_test.go` in our code, but I will call out a few things.

A snippet of one of the test functions is provided below:
 

```go
func TestOnHttpResponseHeaders(t *testing.T) {
   vmTest(t, func(t *testing.T, vm types.VMContext) {
   opt := proxytest.NewEmulatorOption().
           WithPluginConfiguration([]byte(`{"targetURLPrefixes": ["my-host.com"]}`)).
           WithVMContext(vm)
   host, reset := proxytest.NewHostEmulator(opt)
   defer reset()

   require.Equal(t, types.OnPluginStartStatusOK, host.StartPlugin())

   t.Run("http status code is preserved", func(t *testing.T) {
           // Initialize http context.
           id := host.InitializeHttpContext()

           // Call OnHttpRequestHeaders with the headers set to simulate a real request
           hs := [][2]string{{":authority", "my-host.com"}, {":scheme", "https"}, {":path", "/"}}
           action := host.CallOnRequestHeaders(id, hs, false)
```

* Note that the SDK provides an emulator that can be used for our unit tests
* We can provide configuration via `WithPluginConfiguration`
* In our case we will use go test to run our tests instead of using tiny go...mainly because that allows us to use the testify package


At the bottom of the `main_test.go` file we have this snippet of code:

```go
// vmTest executes f twice, once with a types.VMContext that executes plugin code directly
// in the host, and again by executing the plugin code within the compiled main.wasm binary.
// Execution with main.wasm will be skipped if the file cannot be found.
func vmTest(t *testing.T, f func(*testing.T, types.VMContext)) {
        t.Helper()

        t.Run("go", func(t *testing.T) {
                f(t, &vmContext{})
        })

        t.Run("wasm", func(t *testing.T) {
                wasm, err := os.ReadFile("custom-errors.wasm")
                if err != nil {
                        t.Skip("wasm not found")
                }
                v, err := proxytest.NewWasmVMContext(wasm)
                require.NoError(t, err)
                defer v.Close()
                f(t, v)
        })
}
```

You can see that the tests will be called twice one where the code is executed directly and a second time with the compiled binary. So each time you modify the code don't forget to build the binary before running the tests with `go test ./... -v`.

## Deploying our plugin

Next lets create our WASMPlugin kubernetes/Istio resource:

```YAML
apiVersion: extensions.istio.io/v1alpha1
kind: WasmPlugin
metadata:
  name: custom-errors
  namespace: istio-system
spec:
  selector:
    matchLabels:
      app: istio-ingressgateway
  url: oci://myregistry.azurecr.io/istio-custom-errors:v0.0.1
  imagePullPolicy: Always
  imagePullSecret: acr-oci-pull-secret
  phase: AUTHN
  failStrategy: FAIL_OPEN
  pluginConfig:
    problemTypeURIMap:
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
    endStatusCode: 400
    startStatusCode: 599
    targetURLPrefixes:
      - foo.bar.com
```

* This plugin will only be expected for requests received through an Istio Gateway with the label `app: istio-ingressgateway`
* In this case we will package up our binary as an OCI container image. For this to work we must provide a pull secret that can be used to pull images from the registry. This is required even if you have other workloads in the cluster that are able to pull images directly from your cloud provider registry.
* The `phase` field controls where the plugin will be injected
* The `failStrategy` when set to `FAIL_CLOSE` will mean if the WASM binary cannot be accessed or there is an error during its execution the request will be denied. Whereas `FAIL_OPEN` will allow the request to proceed. The documentation advises that plugins that perform some form of Authentication or Authorization should fail close. However, in our case since all we are doing is modifying error responses it is okay and indeed better to fail open. It is important that you have sufficient unit, integration and other types of tests for your plugins to ensure that the plugin will not impact performance or cause an outage.

We are going to use an OCI image to deploy our plugin as this is relatively simple option. However, there are other options:

* `url` can be a `http[s]://` URL - this does mean that this URL must be accessible to Istio without authentication.
* `url` can also refer to a file via `file://` that is local to the proxy container. You could for example do this by storing the binary in a ConfigMap and then mounting this in proxies using the `sidecar.istio.io/userVolume` and `sidecar.istio.io/userVolumeMount` annotations. But to be honest this is not really practical for production.

Our container image can be created from the scratch image:

```Dockerfile
FROM scratch
COPY custom-errors.wasm ./plugin.wasm
ENTRYPOINT [ "plugin.wasm" ]
```

Azure Container Registry (ACR) supports OCI images so in our example we are publishing the image to ACR.

```bash
VERSION="v0.0.1"

ACR_REPO="myregistry"
ACR_DOMAIN="${ACR_REPO}.azurecr.io"
DOCKER_IMAGE="${ACR_DOMAIN}/istio-custom-errors:${VERSION}"

docker build . -t "${DOCKER_IMAGE}"

az acr login -n "${ACR_REPO}"

docker push "${DOCKER_IMAGE}"
```


But we do need to create a pull secret - here we are using an EntraID Service Principal (an alternative would be to use a [repository scoped access token](https://learn.microsoft.com/en-us/azure/container-registry/container-registry-repository-scoped-permissions)).

```bash
ARM_CLIENT_ID=xxxx-xxxx-xxxx-xxxxxxx-xxxx
ARM_CLIENT_SECRET=somesecret

kubectl create secret docker-registry acr-oci-pull-secret \
    --namespace istio-system \
    --docker-server=myregistry.azurecr.io \
    --docker-username="${ARM_CLIENT_ID}" \
    --docker-password="${ARM_CLIENT_SECRET}"
```

Note that to display info level logs you would need to change the default logging configuration. This can be achieved with the istioctl command line tool:

`istioctl proxy-config log istio-ingressgateway-5576bb6d65-qzj9j  --level wasm:info`
