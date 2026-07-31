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
- records load-balancer/member ownership in policy `external_ids`;
- safely shares an identical policy and refuses to overwrite an
  unmanaged or conflicting policy;
- removes ownership during member deletion and restores missing policy
  state during member synchronization.

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

This is a downstream compatibility patch and has not been accepted
upstream.
