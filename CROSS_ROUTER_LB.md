# Cross-router OVN load-balancer return path

This fork extends `ovn-octavia-provider` 10.0.0 for an OVN topology in
which an OVN load balancer VIP and a member are attached to different
logical routers and each router has its own external gateway.

OVN installs the forward DNAT path, but a remote member can return via
its own default gateway instead of the VIP router that owns the
un-DNAT state. The resulting symptom is intermittent timeout, normally
tracking the share of traffic sent to remote-router members.

The provider now:

- registers the OVN Northbound `Logical_Router_Static_Route` and
  `Logical_Router_Policy` tables in its restricted IDL;
- selects the most-specific non-default route from the member router
  toward the VIP;
- installs a priority-31000 source/service reroute policy for a remote
  member;
- installs a member-address `/32` (IPv4) or `/128` (IPv6)
  `policy=src-ip` route toward the selected transit next hop. This
  supplies the return route when a TransitGateway-attached member
  router intentionally has no default route;
- records load-balancer/member ownership in both policy and route
  `external_ids`;
- safely shares an identical policy and refuses to overwrite an
  unmanaged or conflicting policy/route;
- removes ownership and the final route during member deletion, and
  restores missing state during member synchronization.

The runtime overlay image can be built on the exact Octavia image used
by a deployment:

```shell
docker build \
  --build-arg OCTAVIA_BASE_IMAGE='<octavia-image-or-digest>' \
  -f Dockerfile.runtime-patch \
  -t ovn-octavia-provider-cross-router:10.0.0 .
```

In the validating two-router environment, the unmodified provider
produced 16 successful HTTP requests and 14 timeouts. With the managed
return policy, 30 of 30 requests succeeded across both real VM
backends. Member deletion removed the policy and member recreation
restored it.

An additional controller-managed TransitGateway test used two Keystone
projects, two real HTTP VMs, one attachment per VPC, and a cross-project
Octavia member. A reroute policy alone produced an empty `nexthops`
column first (the provider used the wrong OVSDB field) and, after that
was corrected, still lost the remote member's SYN-ACK because its router
had no default route. The source-policy host route fixed that narrower
case: 40/40 requests succeeded (22 local, 18 remote), deletion removed
the route, and recreation restored it with a new member owner; a second
20/20 request run succeeded (11 local, 9 remote).

Whole-load-balancer deletion is covered separately from member deletion:
it removes every ownership token for that LB before detaching/deleting
the OVN load balancer. The live cascade test left zero matching source
routes.

This is a downstream compatibility patch and has not been accepted
upstream.
