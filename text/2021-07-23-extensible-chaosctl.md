# Extensible Chaosctl

## Summary

A tool to control the status of Chaos Mesh as much as possible in an extensible way. 

## Motivation

Currently, the chaosctl is a debug tool to collect logs and other information from several kinds of chaos. 
However, some of its functions are implemented by executing commands in the namespace of target pods through the chaos-daemon.

For example, in the iochaos debug part, the chaosctl executes `cat /proc/mounts` [command](https://github.com/chaos-mesh/chaos-mesh/blob/4b8fb5ba1518fda0d144c8df9239dcb0381ff485/pkg/chaosctl/debug/iochaos/iochaos.go#L54) straightly in the namepsace of target pods, 
which is dangerous and causes develpment difficulties in the long time.

To implement more features for chaosctl, we must refactor it for more security and extensibility.


## Detailed Design

For more security, functions like reading file or listing processes should be implemented in the server side.
In other words, we need an API server, and make the chaosctl a pure client.

So, what APIs do we needs? To implement the features that current chaosctl supports, maybe we can provide restful APIs like `/networkchaos/iptables` or `/iochaos/mounts`.
However, in some cases we may need `/networkchaos/iptables` and in other cases we need `/networkchaos/ipset` or both of them, should we provide API for each of them to avoid unsued information?

> I think the unsued information will reduce the debug efficiency very much.

On the other hand, there are too many kinds of resources related with each other: the scheduler, the xxxchaos, the podcxxxchaos, the pod, the container and the process, how can we provide all APIs with their arrangement?

> For example, how can we provide both of `/xxxchaos/podchaos` and `/podchaos/xxxchaos` conveniently?

To resolve these problems, we need structural APIs.

### Nested Resources

Nested resources, also called subresources, may accomplish our goals. For example, if we register the struture of `networkchaos` resources, its subresources like `iptables` and `ipset` will be registerd automatically. We can access the networkchaos resources by path `/networkchaos` and access its subresources by path `/networkchaos/iptables` or `/networkchaos/ipset`.

```go
type NetWorksChaos struct {
    Iptables []*IptablesRule
    Ipset    []*IpsetRule
}
```

However, there are two main drawbacks of this solution. Firstly, it can not support cascade queries, we must regard `/networkchaos` and `/networkchaos/<name>` as different resources. Secondly, there is almost no library to conveniently build nested API servers in the ecosystem of golang.

### GraphQL

The GraphQL is one of the most famous solutions for structural APIs. In above case, we can easy fetch iptables resources only by query `{ networkchaos { iptables } }`.
However, the main drawback of this solution is that queries are not convenient to edit in cli, we need translation between resource paths and GraphQL queries.

For example, we can translate resource path `/networkchaos/iptables` to query `{ networkchaos { iptables } }`.

### API Server

We can choose one of the nested resources and GraphQL as our API solution, I prefer GraphQL as its ecosystem is better than nested resources.

Moreover, server should provide additional resources like `logs`.

### Client

#### Usage

- identify resources

We can identify target resources by path. For example, the path `/networkchaos/<name>/podchaos/<pod-name>` will identify the podnetworkchaos with `id=<pod-name>` owned by networkchaos with `id=<name>`. If the `id` is ignore and resource type is plural, like `/networkchaoses/podchaoses`, all resources belonging to these kinds will be identified.

- `show` and `desc`

We provide two basic debug subcommands, `get` and `desc`. The `get` command only shows key information like `name` or `id` of resources while `desc` command shows full information.


- `delete`

We will provide `delete` subcommand, witch will delete identified resources.

#### Translation

If we choose GraphQL as the API solution, we must translate paths to queries. The plural subpaths will be tanslated to GraphQL fragments without any parameters.
For example, resource paths `/networkchaoses/iptableses` will be translated to query `{ networkchaos { iptables } }`.

And the cascade subpaths will be tanslate to fragments with paramenters. For example, resource paths `/networkchaos/<name>/iptables/<podName>` will be translated to following query.

```GraphQL
# { "name": "<name>", "podName": "<podName>" }
query GetIptables($name: String!, $podName: String!) {
  networkchaos(name: $name) {
    iptables(name: $podName)
  }
}
```

#### Auto-Completion

We can improve auto-completion with the schema and data. 
For example, when the user types `chaosctl get /network`, we can complete the command to `chaosctl get /networkchaos/` or `chaosctl get /networkchaoses` by schema. Then, if the user choose `chaosctl get /networkchaos/`, we can send query `{ networkchaos { name } }` and complete the command to `chaosctl get /networkchaos/<name>` by the result.

## Alternatives

Implement all APIs one by one as our requirements.