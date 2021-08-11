# Implement Physical Machine Chaos in Chaos Mesh

## Background

Now we have implemented some chaos in [Chaosd](https://github.com/chaos-mesh/chaosd),
which is used to inject fault to the physical machine. It's a simple command-line
tool, and also can work as a server. It lacks UI（just like Chaos Mesh Dashboard
to create and manage experiments, and can't orchestrate the experiment just like
the [workflow](https://chaos-mesh.org/docs/create-chaos-mesh-workflow).

## Proposal

Implement physical machine chaos in Chaos Mesh, and then we can reuse the
Dashboard and workflow for it.

There are two ways to implement it:

### 1. Treat physical machines as a new selector

In this way, we will reuse the chaos type in Chaos Mesh, and implement a new
selector to choose physical machines.

For example, below is a YAML config for NetworkChaos:

```YAML
apiVersion: chaos-mesh.org/v1alpha1
kind: NetworkChaos
metadata:
  name: network-delay
  namespace: busybox
spec:
  action: delay
  mode: all
  selector:
    namespaces:
      - busybox
  delay:
    latency: "5s"
```

It will inject network delay on all pods in the busybox namespace.

For physical machine, we can implement a new selector, the YAML config may looks
like:

```YAML
apiVersion: chaos-mesh.org/v1alpha1
kind: NetworkChaos
metadata:
  name: network-delay
  namespace: chaos-testing
spec:
  action: delay
  selector:
    physicalMachines:
      - 123.123.123.123:123
      - 124.124.124.124:124
  delay:
    latency: "5s"
```

We replace the selector `namespaces` to `physicalMachines` , and the
`123.123.123.123:123` and `124.124.124.124:124` are the addresses of Chaosd server.

#### Advantage of proposal 1

The physics experiment and K8s experiment are unified, the only difference is
the selectors.

#### Disadvantage of proposal 1

Higher implementation costs.

* We know that Chaos Mesh is designed for K8s, there are too many codes coupled
to K8s.
* The chaos type is related to a selector, which means each chaos has only one
target type. For example, DNSChaos inject fault to some containers, NetworkChaos
inject fault to some pods. This means we need to implement the physical machine
selector for every chaos.
* The config and implementation of the same chaos type for K8s and physical
machines are different, like DNSChaos, JVMChaos, etc. It is difficult to unify them.

### 2. Treat chaos on the physical machine as a new chaos type

Implement physical machine chaos as a new chaos type in Chaos Mesh. Add a new
crd named `PhysicalMachineChaos` , the config includes:

* action: the subtypes of `PhysicalMachineChaos`, the action can be `stress-cpu`,
`stress-mem`, `network-delay`, `network-loss` and so on.
* address: the addresses of chaosd server.
* config related to the action: for example, for `stress-cpu` action, we need to
set `load` and `workers`.

Here is the sample YAML config for network delay:

```YAML
apiVersion: chaos-mesh.org/v1alpha1
kind: PhysicalMachineChaos
metadata:
  name: physical-network-delay
  namespace: chaos-testing
spec:
  action: network-delay
  address:
    - http://172.16.112.130:31767
  network-delay:
    device: “ens33”
    hostname: “baidu.com”
    duration: "5s"
```

Here is the sample YAML config for CPU stress:

```YAML
apiVersion: chaos-mesh.org/v1alpha1
kind: PhysicalMachineChaos
metadata:
  name: physical-stress-cpu
  namespace: chaos-testing
spec:
  action: stress-cpu
  address:
    - http://172.16.112.130:31767
  stress-cpu:
    workers: 3
    load:  10
```

#### Advantage of proposal 2

* Experiments for the physical machine are relatively independent, and the
implementation of it has no effect on other chaos.
* Low development cost.

#### Disadvantage of proposal 2

We need to define a lot of chaos subtypes in the `PhysicalMachineChaos` , it's a
bit complicated to define it.

### Summary

I prefer to develop through the second option, keep the physical machine
experiment separate, so it is more flexible.
