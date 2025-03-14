---
reviewers:
- jpbetz
- cheftako
title: Migrate Replicated Control Plane To Use Cloud Controller Manager
linkTitle: "Migrate Replicated Control Plane To Use Cloud Controller Manager"
content_type: task
weight: 250
---

<!-- overview -->

{{< glossary_definition term_id="cloud-controller-manager" length="all" prepend="The cloud-controller-manager is">}}

## Background

As part of the [cloud provider extraction effort](/blog/2019/04/17/the-future-of-cloud-providers-in-kubernetes/),
all cloud specific controllers must be moved out of the `kube-controller-manager`.
All existing clusters that run cloud controllers in the `kube-controller-manager`
must migrate to instead run the controllers in a cloud provider specific
`cloud-controller-manager`.

Leader Migration provides a mechanism in which HA clusters can safely migrate "cloud
specific" controllers between the `kube-controller-manager` and the
`cloud-controller-manager` via a shared resource lock between the two components
while upgrading the replicated control plane. For a single-node control plane, or if
unavailability of controller managers can be tolerated during the upgrade, Leader
Migration is not needed and this guide can be ignored.

Leader Migration can be enabled by setting `--enable-leader-migration` on
`kube-controller-manager` or `cloud-controller-manager`. Leader Migration only
applies during the upgrade and can be safely disabled or left enabled after the
upgrade is complete.

This guide walks you through the manual process of upgrading the control plane from
`kube-controller-manager` with built-in cloud provider to running both
`kube-controller-manager` and `cloud-controller-manager`. If you use a tool to deploy
and manage the cluster, please refer to the documentation of the tool and the cloud
provider for specific instructions of the migration.

## {{% heading "prerequisites" %}}

It is assumed that the control plane is running Kubernetes version N and to be
upgraded to version N + 1. Although it is possible to migrate within the same
version, ideally the migration should be performed as part of an upgrade so that
changes of configuration can be aligned to each release. The exact versions of N and
N + 1 depend on each cloud provider. For example, if a cloud provider builds a
`cloud-controller-manager` to work with Kubernetes 1.24, then N can be 1.23 and N + 1
can be 1.24.

The control plane nodes should run `kube-controller-manager` with Leader Election
enabled, which is the default. As of version N, an in-tree cloud provider must be set
with `--cloud-provider` flag and `cloud-controller-manager` should not yet be
deployed.

The out-of-tree cloud provider must have built a `cloud-controller-manager` with
Leader Migration implementation. If the cloud provider imports
`k8s.io/cloud-provider` and `k8s.io/controller-manager` of version v0.21.0 or later,
Leader Migration will be available. However, for version before v0.22.0, Leader
Migration is alpha and requires feature gate `ControllerManagerLeaderMigration` to be
enabled in `cloud-controller-manager`.

This guide assumes that kubelet of each control plane node starts
`kube-controller-manager` and `cloud-controller-manager` as static pods defined by
their manifests. If the components run in a different setting, please adjust the
steps accordingly.

For authorization, this guide assumes that the cluster uses RBAC. If another
authorization mode grants permissions to `kube-controller-manager` and
`cloud-controller-manager` components, please grant the needed access in a way that
matches the mode.

<!-- steps -->

### Grant access to Migration Lease

The default permissions of the controller manager allow only accesses to their main
Lease. In order for the migration to work, accesses to another Lease are required.

You can grant `kube-controller-manager` full access to the leases API by modifying
the `system::leader-locking-kube-controller-manager` role. This task guide assumes
that the name of the migration lease is `cloud-provider-extraction-migration`.

```shell
kubectl patch -n kube-system role 'system::leader-locking-kube-controller-manager' -p '{"rules": [ {"apiGroups":[ "coordination.k8s.io"], "resources": ["leases"], "resourceNames": ["cloud-provider-extraction-migration"], "verbs": ["create", "list", "get", "update"] } ]}' --type=merge`
```

Do the same to the `system::leader-locking-cloud-controller-manager` role.

```shell
kubectl patch -n kube-system role 'system::leader-locking-cloud-controller-manager' -p '{"rules": [ {"apiGroups":[ "coordination.k8s.io"], "resources": ["leases"], "resourceNames": ["cloud-provider-extraction-migration"], "verbs": ["create", "list", "get", "update"] } ]}' --type=merge`
```

### Initial Leader Migration configuration

Leader Migration optionally takes a configuration file representing the state of
controller-to-manager assignment. At this moment, with in-tree cloud provider,
`kube-controller-manager` runs `route`, `service`, and `cloud-node-lifecycle`. The
following example configuration shows the assignment.

Leader Migration can be enabled without a configuration. Please see
[Default Configuration](#default-configuration) for details.

```yaml
kind: LeaderMigrationConfiguration
apiVersion: controllermanager.config.k8s.io/v1
leaderName: cloud-provider-extraction-migration
resourceLock: leases
controllerLeaders:
  - name: route
    component: kube-controller-manager
  - name: service
    component: kube-controller-manager
  - name: cloud-node-lifecycle
    component: kube-controller-manager
```

Alternatively, because the controllers can run under either controller managers,
setting `component` to `*` for both sides makes the configuration file consistent
between both parties of the migration.

```yaml
# wildcard version
kind: LeaderMigrationConfiguration
apiVersion: controllermanager.config.k8s.io/v1
leaderName: cloud-provider-extraction-migration
resourceLock: leases
controllerLeaders:
  - name: route
    component: *
  - name: service
    component: *
  - name: cloud-node-lifecycle
    component: *
```

On each control plane node, save the content to `/etc/leadermigration.conf`, and
update the manifest of `kube-controller-manager` so that the file is mounted inside
the container at the same location. Also, update the same manifest to add the
following arguments:

- `--enable-leader-migration` to enable Leader Migration on the controller manager
- `--leader-migration-config=/etc/leadermigration.conf` to set configuration file

Restart `kube-controller-manager` on each node. At this moment,
`kube-controller-manager` has leader migration enabled and is ready for the
migration.

### Deploy Cloud Controller Manager

In version N + 1, the desired state of controller-to-manager assignment can be
represented by a new configuration file, shown as follows. Please note `component`
field of each `controllerLeaders` changing from `kube-controller-manager` to
`cloud-controller-manager`. Alternatively, use the wildcard version mentioned above,
which has the same effect.

```yaml
kind: LeaderMigrationConfiguration
apiVersion: controllermanager.config.k8s.io/v1
leaderName: cloud-provider-extraction-migration
resourceLock: leases
controllerLeaders:
  - name: route
    component: cloud-controller-manager
  - name: service
    component: cloud-controller-manager
  - name: cloud-node-lifecycle
    component: cloud-controller-manager
```

When creating control plane nodes of version N + 1, the content should be deployed to
`/etc/leadermigration.conf`. The manifest of `cloud-controller-manager` should be
updated to mount the configuration file in the same manner as
`kube-controller-manager` of version N. Similarly, add `--enable-leader-migration`
and `--leader-migration-config=/etc/leadermigration.conf` to the arguments of
`cloud-controller-manager`.

Create a new control plane node of version N + 1 with the updated
`cloud-controller-manager` manifest, and with the `--cloud-provider` flag set to
`external` for `kube-controller-manager`. `kube-controller-manager` of version N + 1
MUST NOT have Leader Migration enabled because, with an external cloud provider, it
does not run the migrated controllers anymore, and thus it is not involved in the
migration.

Please refer to [Cloud Controller Manager Administration](/docs/tasks/administer-cluster/running-cloud-controller/)
for more detail on how to deploy `cloud-controller-manager`.

### Upgrade Control Plane

The control plane now contains nodes of both version N and N + 1. The nodes of
version N run `kube-controller-manager` only, and these of version N + 1 run both
`kube-controller-manager` and `cloud-controller-manager`. The migrated controllers,
as specified in the configuration, are running under either `kube-controller-manager`
of version N or `cloud-controller-manager` of version N + 1 depending on which
controller manager holds the migration lease. No controller will ever be running
under both controller managers at any time.

In a rolling manner, create a new control plane node of version N + 1 and bring down
one of version N until the control plane contains only nodes of version N + 1.
If a rollback from version N + 1 to N is required, add nodes of version N with Leader
Migration enabled for `kube-controller-manager` back to the control plane, replacing
one of version N + 1 each time until there are only nodes of version N.

### (Optional) Disable Leader Migration {#disable-leader-migration}

Now that the control plane has been upgraded to run both `kube-controller-manager`
and `cloud-controller-manager` of version N + 1, Leader Migration has finished its
job and can be safely disabled to save one Lease resource. It is safe to re-enable
Leader Migration for the rollback in the future.

In a rolling manager, update manifest of `cloud-controller-manager` to unset both
`--enable-leader-migration` and `--leader-migration-config=` flag, also remove the
mount of `/etc/leadermigration.conf`, and finally remove `/etc/leadermigration.conf`.
To re-enable Leader Migration, recreate the configuration file and add its mount and
the flags that enable Leader Migration back to `cloud-controller-manager`.

### Default Configuration

Starting Kubernetes 1.22, Leader Migration provides a default configuration suitable
for the default controller-to-manager assignment.
The default configuration can be enabled by setting `--enable-leader-migration` but
without `--leader-migration-config=`.

For `kube-controller-manager` and `cloud-controller-manager`, if there are no flags
that enable any in-tree cloud provider or change ownership of controllers, the
default configuration can be used to avoid manual creation of the configuration file.

### Special case: migrating the Node IPAM controller {#node-ipam-controller-migration}

If your cloud provider provides an implementation of Node IPAM controller, you should
switch to the implementation in `cloud-controller-manager`. Disable Node IPAM
controller in `kube-controller-manager` of version N + 1 by adding
`--controllers=*,-nodeipam` to its flags. Then add `nodeipam` to the list of migrated
controllers.

```yaml
# wildcard version, with nodeipam
kind: LeaderMigrationConfiguration
apiVersion: controllermanager.config.k8s.io/v1
leaderName: cloud-provider-extraction-migration
resourceLock: leases
controllerLeaders:
  - name: route
    component: *
  - name: service
    component: *
  - name: cloud-node-lifecycle
    component: *
  - name: nodeipam
-   component: *
```

## {{% heading "whatsnext" %}}

- Read the [Controller Manager Leader Migration](https://github.com/kubernetes/enhancements/tree/master/keps/sig-cloud-provider/2436-controller-manager-leader-migration)
  enhancement proposal.

