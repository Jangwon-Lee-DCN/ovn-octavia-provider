# OVN Octavia provider fork Agent contract

Read `/home/ubuntu/AGENTS.md` first. Preserve upstream OpenStack documentation
and release notes. This fork owns only the downstream cross-router load-balancer
patch documented in `CROSS_ROUTER_LB.md` and its tests.

Changes affecting VPC semantics, runtime image, Octavia values, or production
rollout require a central cross-repository change contract and exact tested
revisions. Do not describe downstream behavior as accepted upstream.
