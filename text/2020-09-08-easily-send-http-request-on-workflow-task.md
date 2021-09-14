# Easily Send HTTP Request On Workflow Task

- [Easily Send HTTP Request On Workflow Task](#easily-send-http-request-on-workflow-task)
  - [Summary](#summary)
  - [Motivation](#motivation)
  - [Detailed design](#detailed-design)
    - [Rendering `Task` for sending HTTP request](#rendering-task-for-sending-http-request)
      - [Why `curl`](#why-curl)
      - [Frontend form parameters](#frontend-form-parameters)
      - [Load request body from ConfigMap as file](#load-request-body-from-configmap-as-file)
      - [Advanced configuration](#advanced-configuration)
      - [Frontend component for this type of `Task` and preview of generated result](#frontend-component-for-this-type-of-task-and-preview-of-generated-result)
    - [New context variable: `http`](#new-context-variable-http)
      - [Structure of `http`](#structure-of-http)
      - [Example conditions](#example-conditions)
  - [Drawbacks](#drawbacks)
    - [parsing curl command inline back into the parameters](#parsing-curl-command-inline-back-into-the-parameters)
    - [parser of context variable `http`](#parser-of-context-variable-http)
  - [Alternatives](#alternatives)
    - [Alternative solution 1: New type of WorkflowNode/Template for sending HTTP request](#alternative-solution-1-new-type-of-workflownodetemplate-for-sending-http-request)
  - [Unresolved questions](#unresolved-questions)

## Summary

Rendering `curl` command line into Workflow `Task`  with parameters, with common
used parameters for HTTP. And more useful context variables like `json` or `http`.

> I used to design the rendering logic with pure frontend/typescript, but as the
> requirement of "parsing curl command line", I think using golang is better to
> reuse the codes and round-trip testing.

## Motivation

The design of `Task` node in Workflow is very common-usable, users could set any
workload as one `ContainerSpec` inside the `Task`, and select branches with
conditions. But it's quite complex to use the `Task`, users need to write an
entire one-line shell to call the utilities in docker image, such as `curl -X
GET https://your.application/helath -H 'Required-Token: token'`. And we do not
want to introduce more types of WorkflowNode for doing that, so I think
rendering the original require of "sending HTTP request" to `Task` is a good
idea.

As the certain context of "sending HTTP request", we could introduce some
dedicated context variables only for parsing HTTP status code, and response
body. For most usage, an HTTP endpoint like `https://your.application/health`
will return `20x` as application is healthy, `50x` as application is not
available.

## Detailed design

### Rendering `Task` for sending HTTP request

The basic idea is There are some utilities called "curl command line builder",
we could borrow the core logic from that.

#### Why `curl`

We will raise a pod inside of the kubernetes cluster, so we should consider
about the size of image and multi architecture/platform support.

I will use `curlimages/curl:7.78.0` as the docker image. It's the latest stable
curl for now(2021-09-07). It's small(< 4MB). It supports most common
architectures: `386`, `amd64`, `arm64` and so on.

Why not use other tools like `httpie`?

`httpie` provides more easy flags and better colorful output of HTTP response.
But the image is much larger(about 25MB) and no support for other architectures.

Why not use python/javascript script files with `python` and `node` dockerimage?

Both of official images are large(about 60MB for python and 40MB for node). And
it's much harder for generating codes instead of generating command line.

Why not build another standalone binary tool?

We do not want reinvent another wheel. :D

#### Frontend form parameters

We will provide the most common used parameters on the frontend:

- HTTP method
- URL
- Custom headers
- Request body in string
- Path of the file that as the content of request body

"Request body in string" and "Path of the file that as the content of request body"
are exclusive.

> I used to mount "configmap" into the pod, and use it's content as the request body
> directly. But it's could not be implemented now, because the name of configmap
> would not appear in the command line, so parser could not rebuild this, round-trip
> not works.

#### Load request body from ConfigMap as file

`curl` supports loading content from file as request body with `-d`. We could
load an existed configmap as the content of request body, the mount path of
configmap could be configured.

#### Advanced configuration

And we could provide advanced configurations:

- flags: custom flags that will be appended at the end of command line directly
- image: replace the default image `curlimages/curl:7.78.0`, useful in air-gap
  cluster

#### Frontend component for this type of `Task` and preview of generated result

We want store the state into one or several `annotation` of templates, but it
could not be done in short future. There are no embedded `metav1.ObjectMeta`
inside the `Template`. We'd better to treat `Template` with `WorkflowNode` as
`PodTemplateSpec` to `Pod`. I think we should split CRDs for workflow into
another subgroup, then upgrade to `v1alpha2`.

If we want to show the form of configuration of HTTP request in frontend, we
could only parse the command line of `curl`, and that is awful: it's hard to
keep consistently between original parameters and parsed parameters. I prefer
NOT to do that. But this feature is very important for user to modifying the
exists workflow. So I have to implement this. Welcome to other way to implement
this.

The generated `Task` should show the preview instantly.

### New context variable: `http`

Although we could use `stdout` to collect the output of `curl`, it meaningless
that we do not provide the parsed HTTP context for the next conditional
branches.

So we will provide another context variable called `http` for introducing the
parsed HTTP context. It contains very simplified and common-used HTTP request
and response, easy for being used in the conditions.

#### Structure of `http`

```golang
type ContextVariableHTTP struct {
    Request  HTTPRequest  `json:"request"`
    Response HTTPResponse `json:"response"`
}
type HTTPRequest struct {
    Method string      `json:"method"`
    URL    string      `json:"url"`
    Header http.Header `json:"header"`
    Body   []byte      `json:"body"`
}
type HTTPResponse struct {
    StatusCode int         `json:"statusCode"`
    Header     http.Header `json:"header"`
    Body       []byte      `json:"body"`
}

```

#### Example conditions

Response returns 20x:

`http.response.statusCode >= 200 && http.response.statusCode < 300`

Response returns 50x:

`http.response.statusCode >= 500`

And used could select these fields in context variable `http`, and use it in the
condition.

## Drawbacks

There are several things that bother me a long time, please leave comments if
you have any idea!

### parsing curl command inline back into the parameters

Here is one concerned thing about "parsing curl command inline back into the
parameters".

Origin demand:

- restore the parameters of HTTP request, for the next show and modification on
  workflow view.
- restore the parameters of HTTP request, for the context variable
  `http.request`

Excepted implementation:

Store all the parameters with `annotation`, with specific prefix like
`chaos-mesh.opg/workflow.http.method=GET`

Actual implementations:

Because struct `Template` in WorkflowSpec does not embed `metav1.Object`, there
is nowhere to place the annotations. So we have to parse the rendered "curl
command line". We need a lot of works and tests for keep the parsed result is
consistent with the original parameters.

### parser of context variable `http`

I want to parse the output of the `curl`, with flag `-i`. It will print both response
header and body to `stdout`. So we need a parser, with a lot of testcase about
`curl`'s output and expected `HTTPResponse`.

And about the `-L` and http `301`/`302`, I think only keep the last response is
the right way.

Another thing is the `stdout` context variable is not only contains `stdout`,
but also `stderr`. It might bring confusing for users. But kubernetes does not
provide a way to split `stdout` and `stderr` from the `log` subresource from
`Pod`, we have no idea expected implement collector for each container runtime.
It will not break anything yet, but it does bring a little mess. Maybe we need a
rename, from `stdout` to `log`.

## Alternatives

### Alternative solution 1: New type of WorkflowNode/Template for sending HTTP request

I do really want to add some annotations as the metadata of rendering this Template,
but I will not turn them into the "official" fields of CRD. Current CRD is powerful
enough for describing the behavior.

## Unresolved questions

- This implementation is not so friendly to users with pure command-line and
  yaml files.
