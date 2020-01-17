---
layout: "docs"
page_title: "Connect - WAN Federation via Mesh Gateways"
sidebar_current: "docs-connect-wanfed-meshgateways"
description: |-
  WAN federation via mesh gateways allows for Consul servers in different datacenters to be federated exclusively through mesh gateways.
---

-> **1.8.x+:**  This feature is available in Consul versions 1.8.x **TODO(wanfed)** and newer.

# WAN Federation via Mesh Gateways

~> This topic requires familiarity with [mesh gateways](/docs/connect/mesh_gateways.html).

WAN federation via mesh gateways allows for Consul servers in different datacenters
to be federated exclusively through mesh gateways.

When setting up a
[multi-datacenter](https://learn.hashicorp.com/consul/security-networking/datacenters)
Consul cluster, operators must ensure that all Consul servers in every
datacenter must be directly connectable over their WAN-advertised network
address from each other.

This requires that operators setting up the virtual machines or containers
hosting the servers take additional steps to ensure the necessary routing and
firewall rules are in place to allow the servers to speak to each other over
the WAN.

Sometimes this prerequisite is difficult or undesirable to meet:

* **Difficult:** The datacenters may exist in multiple Kubernetes clusters that
  unfortunately have overlapping pod IP subnets, or may exist in different
  cloud provider VPCs that have overlapping subnets.

* **Undesirable:** Network security teams may not approve of granting so many
  firewall rules. With platform autoscaling in place it may not even be tenable
  to keep the rules up-to-date.

Operators looking to simplify their WAN deployment and minimize the exposed
security surface area can elect to join these datacenters together using [mesh
gateways](/docs/connect/mesh_gateways.html) to do so.

## Architecture

There are two main kinds of communication that occur over the WAN link spanning
the gulf between disparate Consul datacenters:

* **WAN gossip:** We leverage the serf and memberlist libraries to gossip
  around failure detector knowledge about Consul servers in each datacenter.
  By default this operates point to point between servers over `8302/udp` with
  a fallback to `8302/tcp` (which logs a warning indicating the network is
  misconfigured).

* **Cross-datacenter RPCs:** Consul servers expose a special multiplexed port
  over `8300/tcp`. Several distinct kinds of messages can be received on this
  port, such as RPC requests forwarded from servers in other datacenters.

        TODO(wanfed): <NORMAL-DIAGRAM>

        [ name:server1/dc1 ]  <----(wan-gossip  )----> [ name:server3/dc2 ]
        | lan: 10.0.0.1    |                           [ lan: 10.1.2.1    ]
        [ wan: 37.4.5.7    ]  <----(cross-dc RPC)----> [ wan: 54.6.7.9    ]

                /\
                ||
            (lan-gossip )
                ||
            (RPC for dc2)
                ||

        [ name:client9/dc1 ]
        [ lan: 10.0.0.9    ]

In this network topology individual Consul client agents on a LAN in one
datacenter never need to directly dial servers in other datacenters. This
means you could introduce a set of firewall rules prohibiting `10.0.0.0/24`
from sending any traffic at all to `10.1.2.0/24` for security isolation.

You may already have configured [mesh
gateways](https://learn.hashicorp.com/consul/developer-mesh/connect-gateways)
to allow for services in the service mesh to freely connect between datacenters
regardless of the lateral connectivity of the nodes hosting the Consul client
agents.

By activating WAN federation via mesh gateways [TODO(wanfed):link] the servers
can similarly use the existing mesh gateways to reach each other without
themeselves being directly reachable.

        TODO(wanfed): <WANFED-DIAGRAM>

        [ name:gateway1/dc1 ]  <----(wan-gossip  )----> [ name:gateway4/dc2 ]
        | lan: 10.0.0.5     |                           [ lan: 10.1.2.5     ]
        [ wan: 37.4.5.7     ]  <----(cross-dc RPC)----> [ wan: 54.6.7.9     ]

                /\                                             /\
                ||                                             ||
            (wan-gossip )                                  (wan-gossip )
                ||                                             ||
            (cross-dc RPC)                                 (cross-dc RPC)
                ||                                             ||
                \/                                             \/

        [ name:server1/dc1  ]                           [ name: server3/dc2  ]
        [ lan: 10.0.0.1     ]                           [ lan: 10.1.2.1      ]

## Configuration

### TLS

All Consul servers in all datacenters should have TLS configured with certificates containing
these SAN fields:

    server.<this_datacenter>.<domain>              (normal)
    <node_name>.server.<this_datacenter>.<domain>  (needed for wan federation)

[TODO(wanfed):link]

This can be achieved using any number of tools, including `consul tls cert
create` with the `-node-name` flag.

### Mesh Gateways

There needs to be at least one mesh gateway configured to opt-in to exposing
the servers in its configuration. When using the `consul connect envoy` CLI
this is done by using the flag `-expose-servers`. All this does is to register
the mesh gateway into the catalog with the additional piece of service metadata
of `{"consul-wan-federation":"1"}`. If you are registering the mesh gateways
into the catalog out of band you may simply add this to your existing
registration payload.

!> Before activating the feature on an existing cluster you should ensure that
there is at least one mesh gateway prepared to expose the servers registered in
each datacenter otherwise the WAN will become only partly connected.

### Consul Server Options

There are a few necessary additional pieces of configuration beyond those
required for standing up a
[multi-datacenter](https://learn.hashicorp.com/consul/security-networking/datacenters)
Consul cluster.

Consul servers in the _primary_ datacenter should add this snippet to the
configuration file:

```hcl
connect {
  enabled = true
  enable_mesh_gateway_wan_federation = true
}
```

Consul servers in all _secondary_ datacenters should add this snippet to the
configuration file:

```hcl
primary_gateways = [ "<primary-mesh-gateway-ip>:<primary-mesh-gateway-port>", ... ]
connect {
  enabled = true
  enable_mesh_gateway_wan_federation = true
}
```

Any references to `start_join_wan` or `retry_join_wan` should be omitted. [TODO(wanfed):link]

-> The `primary_gateways` configuration can also use `go-discover` syntax just
like `retry_join_wan`.

### Bootstrapping

For ease of debugging (such as avoiding a flurry of misleading error messages)
when intending to activate WAN federation via mesh gateways it is best to
follow this general procedure:

### New secondary

1. Upgrade to the desired version of the consul binary for all servers,
   clients, and CLI.
2. Start all consul servers and clients on the new version in the primary
   datacenter.
3. Ensure the primary datacenter has at least one running, registered mesh gateway with
   the service metadata key of `{"consul-wan-federation":"1"}` set.
4. Ensure you are _prepared_ to launch corresponding mesh gateways in all
   secondaries. When ACLs are enabled actually registering these requires
   upstream connectivity to the primary datacenter to authorize catalog
   registration.
5. Ensure all servers in the primary datacenter have updated configuration and
   restart.
6. Ensure all servers in the secondary datacenter have updated configuration.
7. Start all consul servers and clients on the new version in the secondary
   datacenter.
8. When ACLs are enabled, shortly afterwards it should become possible to
   resolve ACL tokens from the secondary, at which time it should be possible
   to launch the mesh gateways in the secondary datacenter.
   

### Existing secondary

1. Upgrade to the desired version of the consul binary for all servers,
   clients, and CLI.
2. Restart all consul servers and clients on the new version.
3. Ensure each datacenter has at least one running, registered mesh gateway with the
   service metadata key of `{"consul-wan-federation":"1"}` set. 
4. Ensure all servers in the primary datacenter have updated configuration and
   restart.
5. Ensure all servers in the secondary datacenter have updated configuration and
   restart.

### Verification

From any two datacenters joined together double check the following give you an
expected result:

* Check that `consul members -wan` lists all servers in all datacenters with
  their _local_ ip addresses and are listed as `alive`.

* Ensure any API request that activates datacenter request forwarding.  such as
  [`/v1/catalog/services?dc=<OTHER_DATACENTER_NAME>`](/api/catalog.html#dc-1)
  succeeds.