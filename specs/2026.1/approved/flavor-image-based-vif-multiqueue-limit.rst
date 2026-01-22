Flavor/Image-based limit for virtio-net multiqueue
=================================================

Include the URL of your launchpad blueprint:
  https://blueprints.launchpad.net/nova/+spec/flavor-image-based-vif-multiqueue-limit

Introduction
============

Nova supports virtio-net multiqueue, where the number of transmit and
receive queues is automatically set to match the number of guest vCPUs.
While this behavior works well in many scenarios, it can be suboptimal
for instances with a large number of vCPUs.

In particular, OVS-DPDK based environments may suffer from increased CPU
overhead and reduced dataplane efficiency when an excessive number of
queues are created. Operators often need tighter control over queue
allocation to balance performance and resource utilization.

Nova provides a `max_queues` configuration option in `nova.conf` to limit
the number of virtio-net queues. However, this option is configured per
compute node and applies uniformly to all instances running on that node,
making it difficult to tune multiqueue behavior for different instance
sizes or workload characteristics.

This specification proposes adding support for limiting the maximum
number of virtio-net queues using flavor extra specs or image metadata
when multiqueue is enabled.

Problem description
===================

The current virtio-net multiqueue behavior ties the number of queues
directly to the guest vCPU count. For large instances, this can result
in more queues than are practically beneficial, especially in dataplane
heavy workloads using OVS-DPDK.

The existing `max_queues` option in `nova.conf` provides only a coarse
control mechanism, as it applies to all instances on a compute node.
Operators lack a mechanism to express queue limits that vary by instance
type or workload intent.

Use Cases
=========

Operator-controlled instance sizing
-----------------------------------

Operators may want to define different virtio-net multiqueue limits for
different instance types, such as limiting queues for large vCPU flavors
while allowing smaller instances to use the default behavior.

Workload-aware tuning
---------------------

Certain workloads benefit from a limited number of queues due to CPU
pinning, NUMA alignment, or licensing constraints. Flavor or image based
configuration allows these workloads to request appropriate limits
without relying on global configuration changes.

Proposed change
===============

This spec proposes introducing a flavor and image based mechanism to
limit the maximum number of virtio-net queues when multiqueue is enabled.

The limit is specified via flavor extra specs or image metadata and is
fixed at instance creation time. It is not a per-request or per-boot
parameter.

The effective number of virtio-net queues is determined as the minimum
of the following values:

* The number of guest vCPUs
* The limit specified via flavor extra specs or image metadata
* The `max_queues` value defined in `nova.conf`

If no flavor or image limit is specified, Nova preserves the existing
behavior, subject only to the `max_queues` configuration option.

Compute node capability advertisement
-------------------------------------

To represent compute node support for limiting virtio-net multiqueue
queues, a new os-traits capability will be introduced:

* ``COMPUTE_NET_MAX_VIRTIO_MULTIQUEUE``

This trait indicates that the virt driver on the compute node supports
interpreting and enforcing flavor or image based virtio-net multiqueue
limits. Supported virt drivers will advertise this trait, and Nova may
use it to ensure that instances requesting a multiqueue limit are
scheduled only on compatible compute nodes.

Rationale for configuration precedence
--------------------------------------

The precedence order for determining the effective number of virtio-net
queues is defined as follows:

* `nova.conf` configuration
* Flavor extra specs
* Image metadata
* Number of guest vCPUs

This ordering reflects the degree of control and trust associated with
each configuration source from an operator perspective.

The `nova.conf` setting represents an operator-defined upper bound on
each compute node and must always take precedence. Flavor extra specs
are typically managed by administrators under the default Nova policy
(`context_is_admin`) and are therefore considered an operator-approved
mechanism for exposing controlled instance variants.

In contrast, image metadata is generally user-controlled. Under the
default Glance policy, non-admin users can create images and freely set
image properties. Allowing image metadata to override flavor settings
could enable users to request unnecessarily large multiqueue
configurations, leading to inefficient resource usage.

Finally, the number of guest vCPUs acts as a natural upper bound derived
from instance topology but does not convey operator intent regarding
network queue allocation.

This precedence model ensures that operators retain ultimate control
over virtio-net multiqueue behavior while still allowing flexible,
workload-aware tuning via flavors and images.

Alternatives
============

Relying solely on global configuration
-------------------------------------

Using only the existing `max_queues` configuration option was considered
but rejected due to its lack of flexibility in mixed workload
environments.

REST API impact
===============

There are no REST API changes associated with this proposal.

Data model impact
=================

There are no database schema changes required.

Security impact
===============

This change does not introduce new security risks or affect tenant
isolation.

Notifications impact
====================

There are no notification changes.

Other end user impact
=====================

End users may optionally benefit from flavor or image configurations
that provide more appropriate virtio-net multiqueue behavior for their
workloads. Instances that do not use this feature continue to behave as
before.

Performance impact
==================

This change may improve CPU efficiency and dataplane performance for
workloads that benefit from reduced virtio-net queue counts. There is
no performance impact for instances that do not specify a limit.

Developer impact
================

This change requires updates to multiple repositories:

* Nova:
 - Honor flavor and image based virtio-net multiqueue limits
 - Enforce configuration precedence
 - Add unit and functional test coverage

* os-traits:
 - Introduce a new trait representing support for virtio-net multiqueue
   queue limiting

Upgrade impact
==============

There is no upgrade impact. Existing deployments continue to operate
unchanged unless the new configuration options are explicitly used.

References
==========

* Launchpad blueprint: ``flavor-image-based-vif-multiqueue-limit``
