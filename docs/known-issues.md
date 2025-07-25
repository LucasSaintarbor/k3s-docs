---
title: Known Issues
---
The Known Issues are updated periodically and designed to inform you about any issues that may not be immediately addressed in the next upcoming release.
For the most up-to-date information, check open and pinned issues on the [K3s Project Issue Tracker](https://github.com/k3s-io/k3s/issues).
If you are not running the most recent release of K3s, make sure to also search closed issues and release notes to ensure that your issue has not already been resolved.

### Kine/SQL 2147483647 (MAX INT) Revision Limit
When using K3s with an [external SQL database](datastore/ha.md) that was created on a release of K3s prior to May 2024, you must apply schema migrations to your database to allow for more than 2.1 million revisions to be stored within the database.
Databases without updated schema will become read-only when the current datastore revision reaches 2147483647.

You can check the current revision of your datastore by examining the `resourceVersion` field of the response to a list call made against the Kubernetes API.
For example, in the following output, the current revision is 12345:
```bash
$ kubectl get --raw /api/v1/namespaces?labelSelector=none
{"kind":"NamespaceList","apiVersion":"v1","metadata":{"resourceVersion":"12345"},"items":[]}
```

You can update the datastore schema by setting the `KINE_SCHEMA_MIGRATION` environment variable to 1 or higher in the K3s service's env file, and restarting the service.
This change should be made on all servers within the cluster.

### Docker Snap

If you plan to use K3s with the Docker container runtime, the Docker snap package is not recommended as it has been known to cause issues running K3s. Install Docker using the native package management system provided by your operating system.

### iptables

If your node uses iptables v1.6.1 or older in nftables mode you might encounter issues. We recommend utilizing newer iptables (such as 1.6.1+), or running iptables legacy mode, to avoid issues.

```bash
update-alternatives --set iptables /usr/sbin/iptables-legacy
update-alternatives --set ip6tables /usr/sbin/ip6tables-legacy
```

Iptables versions 1.8.0-1.8.4 also have known issues that can cause K3s to fail. Several popular Linux distributions ship with these versions by default. One bug causes the accumulation of duplicate rules, which negatively affects the performance and stability of the node. See [Issue #3117](https://github.com/k3s-io/k3s/issues/3117) for information on how to determine if you are affected by this problem.

K3s includes a known-good version of iptables (v1.8.8) which has been tested to function properly. You can tell K3s to use its bundled version of iptables by starting K3s with the `--prefer-bundled-bin` option, or by uninstalling the iptables/nftables packages from your operating system.

:::info Version Gate

The `--prefer-bundled-bin` flag is available starting with the 2022-12 releases (v1.26.0+k3s1, v1.25.5+k3s1, v1.24.9+k3s1, v1.23.15+k3s1).

:::

### Rootless Mode

Running K3s with Rootless mode is experimental and has several [known issues.](./advanced.md#known-issues-with-rootless-mode)

### Upgrading Hardened Clusters from v1.24.x to v1.25.x {#hardened-125}

Kubernetes removed PodSecurityPolicy from v1.25 in favor of Pod Security Standards. You can read more about PSS in the [upstream documentation](https://kubernetes.io/docs/concepts/security/pod-security-standards/). For K3S, there are some manual steps that must be taken if any `PodSecurityPolicy` has been configured on the nodes.

1. On all nodes, update the `kube-apiserver-arg` value to remove the `PodSecurityPolicy` admission-plugin. Add the following arg value instead: `'admission-control-config-file=/var/lib/rancher/k3s/server/psa.yaml'`, but do NOT restart or upgrade K3S yet. Below is an example of what a configuration file might look like after this update for the node to be hardened:
```yaml
protect-kernel-defaults: true
secrets-encryption: true
kube-apiserver-arg:
  - 'admission-control-config-file=/var/lib/rancher/k3s/server/psa.yaml'
  - 'audit-log-path=/var/lib/rancher/k3s/server/logs/audit.log'
  - 'audit-policy-file=/var/lib/rancher/k3s/server/audit.yaml'
  - 'audit-log-maxage=30'
  - 'audit-log-maxbackup=10'
  - 'audit-log-maxsize=100'
kube-controller-manager-arg:
  - 'terminated-pod-gc-threshold=10'
  - 'use-service-account-credentials=true'
kubelet-arg:
  - 'streaming-connection-idle-timeout=5m'
```
2. Create the `/var/lib/rancher/k3s/server/psa.yaml` file with the following contents. You may want to exempt more namespaces as well. The below example exempts `kube-system` (required), `cis-operator-system` (optional, but useful for when running security scans through Rancher), and `system-upgrade` (required if doing [Automated Upgrades](./upgrades/automated.md)).
```yaml
apiVersion: apiserver.config.k8s.io/v1
kind: AdmissionConfiguration
plugins:
- name: PodSecurity
  configuration:
    apiVersion: pod-security.admission.config.k8s.io/v1beta1
    kind: PodSecurityConfiguration
    defaults:
      enforce: "restricted"
      enforce-version: "latest"
      audit: "restricted"
      audit-version: "latest"
      warn: "restricted"
      warn-version: "latest"
    exemptions:
      usernames: []
      runtimeClasses: []
      namespaces: [kube-system, cis-operator-system, system-upgrade]
```
3. Perform the upgrade as normal. If doing [Automated Upgrades](./upgrades/automated.md), ensure that the namespace where the `system-upgrade-controller` pod is running in is setup to be privileged in accordance with the [Pod Security levels](https://kubernetes.io/docs/concepts/security/pod-security-admission/#pod-security-levels):
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: system-upgrade
  labels:
    # This value must be privileged for the controller to run successfully.
    pod-security.kubernetes.io/enforce: privileged
    pod-security.kubernetes.io/enforce-version: v1.25
    # We are setting these to our _desired_ `enforce` level, but note that these below values can be any of the available options.
    pod-security.kubernetes.io/audit: privileged
    pod-security.kubernetes.io/audit-version: v1.25
    pod-security.kubernetes.io/warn: privileged
    pod-security.kubernetes.io/warn-version: v1.25
```
4. After the upgrade is complete, remove any remaining PSP resources from the cluster. In many cases, there may be PodSecurityPolicies and associated RBAC resources in custom files used for hardening within `/var/lib/rancher/k3s/server/manifests/`. Remove those resources and k3s will update automatically. Sometimes, due to timing, some of these may be left in the cluster, in which case you will need to delete them manually. If the [Hardening Guide](./security/hardening-guide.md) was previously followed, you should be able to delete them via the following:
```sh
# Get the resources associated with PSPs
$ kubectl get roles,clusterroles,rolebindings,clusterrolebindings -A | grep -i psp

# Delete those resources:
$ kubectl delete clusterrole.rbac.authorization.k8s.io/psp:restricted-psp clusterrole.rbac.authorization.k8s.io/psp:svclb-psp clusterrole.rbac.authorization.k8s.io/psp:system-unrestricted-psp clusterrolebinding.rbac.authorization.k8s.io/default:restricted-psp clusterrolebinding.rbac.authorization.k8s.io/system-unrestricted-node-psp-rolebinding && kubectl delete -n kube-system rolebinding.rbac.authorization.k8s.io/svclb-psp-rolebinding rolebinding.rbac.authorization.k8s.io/system-unrestricted-svc-acct-psp-rolebinding
```
