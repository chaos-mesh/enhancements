# Selector Plugin

## Summary

In current implementation, all chaos types are equiped with a selector. E.g. you
have to use `ContainerSelector` in a `PodChaos` though sometimes the
`containerNames` can be ommited and only `PodSelector` is used.

In this RFC, we will split the whole Chaos Mesh definition into three parts:

1. Scope limitation: `Mode` and `Value` works as a scope limitation. It should
   be a standalone component works for all selectors, but not only for
   `PodSelector`.

2. Selector: For example, the `PodSelector`, `AWSSelector`, `GCPSelector` and
   `ContainerSelector`. They should be able to list all selected resource.

3. Implementation: The specific chaos implementation, which has been extracted
   from the controller routine.

However, the first two parts are hard coupling now. We will firstly define their
interface, and then describe how could we develop a new selector plugin.

## Routine

When the controller receives a request, it will do following things:

1. Call related selectors to list all selected resources.
2. Call scope limitation to randomly choose some of them.
3. Iterate through the selected resources and call the implementation.

## Interface

### Scope limitation

The scope limitation is quite simple now. It accepts a list and return a list.
As all records Ids are strings, the only method is `Limit(input []string)
[]string`. This interface could be unstable as we will not design the "scope
limitation plugin" in the near future, so that we could add more functions to it
when needed.

For example, one day if we need to dynamically add new records, we only need to
add arguments or methods to this interface.

### Selector

One selector should implement this interface:

```go
type RawSelector map[string]interface{}

type Selector interface {
    Validate(selector RawSelector) field.ErrorList
    Default(root map[string]interface{}, selector RawSelector) RawSelector

    Select(selector RawSelector) ([]string, error)
}
```

The controller will pass the raw definition of a selector to the selector
plugin.

#### Definition

At least, they should be declared in the kubernetes object, so we need to
integrate a fully customizable component in the existing resources. The
`unstructured.Unstructured` could be used to represent this situation.

However, it cannot be the embedded fields. For most situation, it is fine as the
`PodSelector` has a `Selector` field, which could be modified to the
`unstructured` type. However, for the `AWSSelector` and `GCPSelector`, we have
to break the compatibility.

It's not a wise choice to bring an unstructured type in the definition, and the
`unstructured.Unstructured` was designed to serve for the full unstructured
type, but not partially unstructured object. But we have no choice.

#### Communication

`grpc` is one possible choice. They could communicate through tcp or unix
socket. The plugin user may need to submit a `Selector` custom resource to
register the selector plugin:

```yaml
apiVersion: chaos-mesh.org/v1alpha1
kind: Selector
metadata:
  name: pod-selector
spec:
  serviceName: xxxx
  ...
```

And then, the controller could establish a `grpc` connection to the service and
hold a selector client.

Other RPC like jsonrpc or http could also be considered, but I prefer grpc as it
has a richer ecosystem in golang's world.

Besides the network communication, chaos-mesh should also provide a way to
communicate through local unix socket, to avoid uneccessary network overhead.
For example, the plugin can be registered with a path:

```yaml
apiVersion: chaos-mesh.org/v1alpha1
kind: Selector
metadata:
  name: pod-selector
spec:
  localPath: /xxxx
  ...
```

### Bind selector with the implementation

The selector should also provide the information about its output: it selected a
pod or a container or a service or a physical machine? The controller should
verify whether the selector and the implementation are compatible.

The type of a selector can be represented in the plugin definition object, for
example:

```yaml
apiVersion: chaos-mesh.org/v1alpha1
kind: Selector
metadata:
  name: pod-selector
spec:
  type: pod
  ...
```

The `type: pod` means that the output of this selector should be a pod, and
follows the format: `NAMESPACE/NAME`. We should write detailed documents about
every possible formats: like pod, container, physical machine.

The user should also specify the name of the selector he used in the chaos
definition. For example:

```yaml
apiVersion: chaos-mesh.org/v1alpha1
kind: PodChaos
spec:
    selectorName: pod-selector
    selector:
        labelSelector:
    ...
```

The `selectorName` could have a default value, for different kinds of chaos.
