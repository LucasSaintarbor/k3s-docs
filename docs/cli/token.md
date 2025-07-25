---
title: token
---

# k3s token

K3s uses tokens to secure the node join process and to encrypt confidential information that is persisted to the datastore. Tokens authenticate the cluster to the joining node, and the node to the cluster.

## Token Format

K3s tokens can be specified in either secure or short format. The secure format is preferred, as it enables the client to authenticate the identity of the cluster it is joining, before sending credentials.

### Secure

The secure token format (occasionally referred to as a "full" token) contains the following parts:

`<prefix><cluster CA hash>::<credentials>`

* `prefix`: a fixed `K10` prefix that identifies the token format
* `cluster CA hash`: The hash of the cluster's server CA certificate, used to authenticate the server to the joining node.  
  * For self-signed CA certificates, this is the SHA256 sum of the PEM-formatted certificate, as stored on disk.  
  * For custom CA certificates, this is the SHA256 sum of the DER encoding of the root certificate; commonly known as the certificate fingerprint.
* `credentials`: The username and password, or bearer token, used to authenticate the joining node to the cluster.

#### TLS Bootstrapping

When a secure token is specified, the joining node performs the following steps to validate the identity of the server it has connected to, before transmitting credentials:
1. With TLS verification disabled, download the CA bundle from `/cacerts` on the server it is joining.
2. Calculate the SHA256 hash of the CA certificate, as described above.
3. Compare the calculated SHA256 hash to the hash from the token.
4. If the hash matches, validate that the certificate presented by the server can be validated by the server's CA bundle.
5. If the server certificate is valid, present credentials to join the cluster using either basic or bearer token authentication, depending on the token type.

### Short

The short token format includes only the password or bearer token used to authenticate the joining node to the cluster.

If a short token is used, the joining node implicitly trusts the CA bundle presented by the server; steps 2-4 in the TLS Bootstrapping process are skipped. The initial connection may be vulnerable to [man-in-the-middle](https://en.wikipedia.org/wiki/Man-in-the-middle_attack) attack.

## Token Types

K3s supports three types of tokens. Only the server token is available by default; additional token types must be configured or created by the administrator.

Type      | CLI Option      | Environment Variable
--------- | --------------- | --------------------
Server    | `--token`       | `K3S_TOKEN`
Agent     | `--agent-token` | `K3S_AGENT_TOKEN`
Bootstrap | `n/a`           | `n/a`

### Server

If no token is provided when starting the first server in the cluster, one is created with a random password. The server token is always written to `/var/lib/rancher/k3s/server/token`, in secure format.

The server token can be used to join both server and agent nodes to the cluster. Anyone with access to the server token essentially has full administrator access to the cluster. This token should be guarded carefully.

The server token is also used as the [PBKDF2](https://en.wikipedia.org/wiki/PBKDF2) passphrase to encrypt confidential information that is persisted to the datastore known as bootstrap data. Bootstrap data is essential to set up new server nodes or restore from a snapshot. For this reason, the token must be backed up alongside the cluster datastore itself.

:::warning
Unless custom CA certificates are in use, only the short (password-only) token format can be used when starting the first server in the cluster. This is because the cluster CA hash cannot be known until after the server has generated the self-signed cluster CA certificates.
:::

For more information on using custom CA certificates, see the [`k3s certificate` documentation](./certificate.md).  
For more information on backing up your cluster, see the [Backup and Restore](../datastore/backup-restore.md) documentation.

### Agent

By default, the agent token is the same as the server token. The agent token can be set before or after the cluster has been started, by changing the CLI option or environment variable on all servers in the cluster. The agent token is similar to the server token in that is it statically configured, and does not expire.

The agent token is written to `/var/lib/rancher/k3s/server/agent-token`, in secure format. If no agent token is specified, this file is a link to the server token.

### Bootstrap

K3s supports dynamically generated, automatically expiring agent [bootstrap tokens](https://kubernetes.io/docs/reference/access-authn-authz/bootstrap-tokens/).

## k3s token

The k3s token CLI tool handles:
* The life cycle of bootstrap tokens, using the same generation and validation code as `kubeadm token` bootstrap tokens. Note that both CLIs are similar.
* The rotation of the server token

```
NAME:
   k3s token - Manage tokens

USAGE:
   k3s token command [command options] [arguments...]

COMMANDS:
   create    Create bootstrap tokens on the server
   delete    Delete bootstrap tokens on the server
   generate  Generate and print a bootstrap token, but do not create it on the server
   list      List bootstrap tokens on the server
   rotate    Rotate original server token with a new server token

OPTIONS:
   --help, -h  show help
```

#### `k3s token create [token]`

Create a new token. The `[token]` is the actual token to write, as generated by `k3s token generate`. If no token is given, a random one will be generated.

A token in secure format, including the cluster CA hash, will be written to stdout. The output of this command should be saved, as the secret portion of the token cannot be shown again.

Flag | Description
---- | ----
`--data-dir` value    | Folder to hold state (default: /var/lib/rancher/k3s or $\{HOME\}/.rancher/k3s if not root)
`--kubeconfig` value  | Server to connect to [$KUBECONFIG]
`--description` value | A human friendly description of how this token is used
`--groups` value      | Extra groups that this token will authenticate as when used for authentication. (default: Default: "system:bootstrappers:k3s:default-node-token")
`--ttl` value         | The duration before the token is automatically deleted (e.g. 1s, 2m, 3h). If set to '0', the token will never expire (default: 24h0m0s)
`--usages` value      | Describes the ways in which this token can be used. (default: "signing,authentication")

#### `k3s token delete`

Delete one or more tokens. The full token can be provided, or just the token ID.

Flag | Description
---- | ----
`--data-dir` value   | Folder to hold state (default: /var/lib/rancher/k3s or $\{HOME\}/.rancher/k3s if not root)
`--kubeconfig` value |Server to connect to [$KUBECONFIG]

#### `k3s token generate`

Generate a randomly-generated bootstrap token.

You don't have to use this command in order to generate a token. You can do so yourself as long as it is in the format `[a-z0-9]{6}.[a-z0-9]{16}`, where the first portion is the token ID, and the second portion is the secret.

Flag | Description
---- | ----
`--data-dir` value   | Folder to hold state (default: /var/lib/rancher/k3s or $\{HOME\}/.rancher/k3s if not root)
`--kubeconfig` value | Server to connect to [$KUBECONFIG]

#### `k3s token list`

List bootstrap tokens, showing their ID, description, and remaining time-to-live.

Flag | Description
---- | ----
`--data-dir` value   | Folder to hold state (default: /var/lib/rancher/k3s or $\{HOME\}/.rancher/k3s if not root)
`--kubeconfig` value | Server to connect to [$KUBECONFIG]
`--output` value     | Output format. Valid options: text, json (default: "text")

#### `k3s token rotate`

:::info Version Gate
Available as of the October 2023 releases (v1.28.2+k3s1, v1.27.7+k3s1, v1.26.10+k3s1, v1.25.15+k3s1).
:::

Rotate original server token with a new server token. After running this command, all servers and any agents that originally joined with the old token must be restarted with the new token.

If you do not specify a new token, one will be generated for you.

 Flag | Description                                                                                
 ---- | ----
`--data-dir` value   | Folder to hold state (default: /var/lib/rancher/k3s or $\{HOME\}/.rancher/k3s if not root)
`--kubeconfig` value | Server to connect to [$KUBECONFIG]     
`--server` value     | Server to connect to (default: "https://127.0.0.1:6443") [$K3S_URL]                                                    
`--token` value      | Existing token used to join a server or agent to a cluster [$K3S_TOKEN]                   
`--new-token` value  | New token that replaces existing token   

:::warning
Snapshots taken before the rotation will require the old server token when restoring the cluster
:::
