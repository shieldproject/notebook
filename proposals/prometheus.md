A Prometheus Exporter for SHIELD
================================

This proposal sets forth the design of a metrics and health status
exporter for the popular [Prometheus Monitoring System][prom].

Why Prometheus?
---------------

It is popular, and it is the default monitoring solution supported
by the Genesis Community.

Metrics to Export
-----------------

Operators are interested primarily in usage of the backup
solution, including archive and storage footprint.

We propose the following metrics:

  1. `tenants_total` - How many Tenants exist.

  1. `agents_total` - How many SHIELD Agents have been registered.

  1. `targets_total` - How many Target Systems have been defined.

  1. `stores_total` - How many Cloud Storage Systems have been
     defined.

  1. `jobs_total` - How many backup jobs have been defined.

  1. `tasks_total` - How many tasks have been created.

  1. `archives_total` - How many Backup Archives have been
     generated.

  1. `storage_used_bytes` - How much storage has been used, in
     bytes.  This metric is tagged with the tenant, target, and
     store that the subset of archives is unique to.

These metrics will all be prefixed with `NAMESPACE_`, where
`NAMESPACE` is configurable by the SHIELD operator at deployment
time.  It defaults to `shield`, yielding metric names like
`shield_tenants_total`.


Health Statii to Track
----------------------

  1. `health_status` - A generic health status, that is
     differentiated into different contexts based on labels.
     For example, the label `target=x tenant=y` means that the
     metric applies to the health of a given target, owned by a
     tenant.

  1. `core_status` - The global initialized / locked status of
     the SHIELD core; (0=uninitialized, 1=locked, 2=unlocked).

These metrics will all be prefixed with `NAMESPACE_`, where
`NAMESPACE` is configurable by the SHIELD operator at deployment
time.  It defaults to `shield`, yielding metric names like
`shield_health_status`.


[prom]: https://prometheus.io/
