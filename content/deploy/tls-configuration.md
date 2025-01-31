+++
date = "2017-03-20T22:25:17+11:00"
title = "TLS Configuration"
weight = 2
[menu.main]
    parent = "security"
+++

Connections between Dgraph database and its clients can be secured using TLS. In
addition, Dgraph can now secure gRPC communications between Dgraph Alpha and
Dgraph Zero server nodes using mutual TLS (mTLS). Dgraph can now also secure
communications over the Dgraph Zero `gRPC-external-private` port used by
Dgraph's Live Loader and Bulk Loader clients. To learn more about the HTTP and
gRPC ports used by Dgraph Alpha and Dgraph Zero, see [Ports Usage]({{< relref "deploy/ports-usage" >}}).
Password-protected private keys are **not supported**.

To further improve TLS security, only TLS v1.2 cypher suites that use 128-bit or
greater RSA or AES encryption are supported.

{{% notice "tip" %}}If you're generating encrypted private keys with `openssl`,
be sure to specify the encryption algorithm explicitly (like `-aes256`). This will
force `openssl` to include `DEK-Info` header in private key, which is required
to decrypt the key by Dgraph. When default encryption is used, `openssl` doesn't
write that header and key can't be decrypted.{{% /notice %}}

## Dgraph Certificate Management Tool

{{% notice "note" %}}
This section refers to the `dgraph cert` command which was introduced in v1.0.9.
For previous releases, see the previous [TLS configuration documentation](https://dgraph.io/docs/v1.0.7/deploy/#tls-configuration).
{{% /notice %}}

The `dgraph cert` program creates and manages CA-signed certificates and private
keys using a generated Dgraph Root CA. There are three types of certificate/key
pairs:
1. Root CA certificate/key pair: This is used to sign and verify node and client
   certificates. If the root CA certificate is changed then you must regenerate
   all certificates, and this certificate must be accessible to the Alpha nodes.
2. Node certificate/key pair: This is shared by the Dgraph Alpha nodes and used
   for accepting TLS connections.
3. Client certificate/key pair: This is used by the clients (like live loader
   and Ratel) to communicate with Dgraph Alpha server nodes where client
   authentication with mTLS is required.

```sh
# To see the available flags.
$ dgraph cert --help

# Create Dgraph Root CA, used to sign all other certificates.
$ dgraph cert

# Create node certificate and private key
$ dgraph cert -n localhost

# Create client certificate and private key for mTLS (mutual TLS)
$ dgraph cert -c dgraphuser

# Combine all in one command
$ dgraph cert -n localhost -c dgraphuser

# List all your certificates and keys
$ dgraph cert ls
```

The default location where the _cert_ command stores certificates (and keys) is
`tls` under the Dgraph working directory. The default directory path can be overridden
using the `--dir` option. For example:

```sh
$ dgraph cert --dir ~/mycerts
```

### File naming conventions

The following file naming conventions are used by Dgraph for proper TLS setup.

| File name | Description | Use |
|-----------|-------------|-------|
| ca.crt | Dgraph Root CA certificate | Verify all certificates |
| ca.key | Dgraph CA private key | Validate CA certificate |
| node.crt | Dgraph node certificate | Shared by all nodes for accepting TLS connections |
| node.key | Dgraph node private key | Validate node certificate |
| client._name_.crt | Dgraph client certificate | Authenticate a client _name_ |
| client._name_.key | Dgraph client private key | Validate _name_ client certificate |

For client authentication, each client must have their own certificate and key.
These are then used to connect to the Dgraph server nodes.

The node certificate `node.crt` can support multiple node names using multiple
host names and/or IP address. Just separate the names with commas when
generating the certificate.

```sh
$ dgraph cert -n localhost,104.25.165.23,dgraph.io,2400:cb00:2048:1::6819:a417
```

{{% notice "tip" %}}You must delete the old node cert and key before you can generate a new pair.{{% /notice %}}

{{% notice "note" %}}When using host names for node certificates, including _localhost_, your clients must connect to the matching host name -- such as _localhost_ not 127.0.0.1. If you need to use IP addresses, then add them to the node certificate.{{% /notice %}}

### Certificate inspection

The command `dgraph cert ls` lists all certificates and keys in the `--dir`
directory (default `dgraph-tls`), along with details to inspect and validate
cert/key pairs.

Example of command output:

```sh
-rw-r--r-- ca.crt - Dgraph Root CA certificate
        Issuer: Dgraph Labs, Inc.
           S/N: 043c4d8fdd347f06
    Expiration: 02 Apr 29 16:56 UTC
SHA-256 Digest: 4A2B0F0F 716BF5B6 C603E01A 6229D681 0B2AFDC5 CADF5A0D 17D59299 116119E5

-r-------- ca.key - Dgraph Root CA key
SHA-256 Digest: 4A2B0F0F 716BF5B6 C603E01A 6229D681 0B2AFDC5 CADF5A0D 17D59299 116119E5

-rw-r--r-- client.admin.crt - Dgraph client certificate: admin
        Issuer: Dgraph Labs, Inc.
     CA Verify: PASSED
           S/N: 297e4cb4f97c71f9
    Expiration: 03 Apr 24 17:29 UTC
SHA-256 Digest: D23EFB61 DE03C735 EB07B318 DB70D471 D3FE8556 B15D084C 62675857 788DF26C

-rw------- client.admin.key - Dgraph Client key
SHA-256 Digest: D23EFB61 DE03C735 EB07B318 DB70D471 D3FE8556 B15D084C 62675857 788DF26C

-rw-r--r-- node.crt - Dgraph Node certificate
        Issuer: Dgraph Labs, Inc.
     CA Verify: PASSED
           S/N: 795ff0e0146fdb2d
    Expiration: 03 Apr 24 17:00 UTC
         Hosts: 104.25.165.23, 2400:cb00:2048:1::6819:a417, localhost, dgraph.io
SHA-256 Digest: 7E243ED5 3286AE71 B9B4E26C 5B2293DA D3E7F336 1B1AFFA7 885E8767 B1A84D28

-rw------- node.key - Dgraph Node key
SHA-256 Digest: 7E243ED5 3286AE71 B9B4E26C 5B2293DA D3E7F336 1B1AFFA7 885E8767 B1A84D28
```

Important points:

* The cert/key pairs should always have matching SHA-256 digests. Otherwise, the cert(s) must be
  regenerated. If the Root CA pair differ, all cert/key must be regenerated; the flag `--force`
  can help.
* All certificates must pass Dgraph CA verification.
* All key files should have the least access permissions, especially the `ca.key`, but be readable.
* Key files won't be overwritten if they have limited access, even with `--force`.
* Node certificates are only valid for the hosts listed.
* Client certificates are only valid for the named client/user.

## TLS options

Starting in release v21.03, pre-existing TLS configuration options have been
replaced by the `--tls` [superflag]({{< relref "deploy/cli-command-reference.md" >}}) 
and its options. The following `--tls` configuration options are available for
Dgraph Alpha and Dgraph Zero nodes:

* `ca-cert <path>` - Path and filename of the Dgraph Root CA (for
   example, `ca.crt`)
* `server-cert <path>` - Path and filename of the node certificate (for
   example, `node.crt`)
* `server-key <path>` - Path and filename of the node certificate private
   key (for example, `node.key`)
* `use-system-ca` - Include System CA with Dgraph Root CA.
* `client-auth-type <string>` - TLS client authentication used to validate client
  connections from external ports. To learn more, see
  [Client Authentication Options](#client-authentication-options).

{{% notice "note" %}}
Dgraph now allows you to specify the path and filename of the CA root
certificate, the node certificate, and the node certificate private key. So,
these files do not need to have specific filenames or exist in the same
directory, as in previous Dgraph versions that used the `--tls_dir` flag.
{{% /notice %}}

You can configure Dgraph Live Loader with the following `--tls` options:

* `ca-cert <path>` - Dgraph root CA, such as `./tls/ca.crt`
* `use-system-ca` - Include System CA with Dgraph Root CA.
* `client-cert` - User cert file provided by the client to Alpha
* `client-key` - User private key file provided by the client to Alpha
* `server-name <string>` - Server name, used for validating the server's TLS host name.


### Using TLS with only external ports encrypted

To encrypt communication between Dgraph server nodes and clients over external
ports, you can configure certificates and run Dgraph Alpha and Dgraph Zero using
the following commands:

Dgraph Alpha:

```sh
# First, create the root CA, Alpha node certificate and private keys, if not already created.
# Note that you must specify in node.crt the host name or IP addresses that clients use connect:
$ dgraph cert -n localhost,104.25.165.23,104.25.165.25,104.25.165.27
# Set up Dgraph Alpha nodes using the following default command (after generating certificates and private keys)
$ dgraph alpha --tls "ca-cert=/dgraph-tls/ca.crt; server-cert=/dgraph-tls/node.crt; server-key=/dgraph-tls/node.key"
```

Dgraph Zero:

```sh
# First, copy the root CA, node certificates and private keys used to set up Dgraph Alpha (above) to the Dgraph Zero node.
# Optionally, you can generate and use a separate Zero node certificate, where you specify the host name or IP addresses used by Live Loader and Bulk Loader to connect to Dgraph Zero.
# Next, set up Dgraph Zero nodes using the following default command:
$ dgraph zero --tls "ca-cert=/dgraph-tls/ca.crt; server-cert=/dgraph-tls/node.crt; server-key=/dgraph-tls/node.key"
```

You can then run Dgraph Live Loader on a Dgraph Alpha node using the following command:

```sh
# Now, connect to server using TLS
$ dgraph live --tls "ca-cert=./dgraph-tls/ca.crt; server-name=localhost" -s 21million.schema -f 21million.rdf.gz
```

### Using TLS with internal and external ports encrypted

If you require client authentication (mutual TLS, or mTLS), you can configure
certificates and run Dgraph Alpha and Dgraph Zero with settings that encrypt
both internal ports (those used within the cluster) as well as external ports
(those used by clients that connect to the cluster, including Bulk Loader and
Live Loader).

The following example shows how to encrypt both internal and external ports:

Dgraph Alpha:

```sh
# First create the root CA, node certificates and private keys, if not already created.
# Note that you must specify the host name or IP address for other nodes that will share node.crt.
$ dgraph cert -n localhost,104.25.165.23,104.25.165.25,104.25.165.27
# Set up Dgraph Alpha nodes using the following default command (after generating certificates and private keys)
$ dgraph alpha
      --tls "ca-cert=/dgraph-tls/ca.crt; server-cert=/dgraph-tls/node.crt; server-key=/dgraph-tls/node.key;
internal-port=true; client-cert=/dgraph-tls/client.alpha1.crt; client-key=/dgraph-tls/client.alpha1.key"
```

Dgraph Zero:

```sh
# First, copy the certificates and private keys used to set up Dgraph Alpha (above) to the Dgraph Zero node.
# Next, set up Dgraph Zero nodes using the following default command:
$ dgraph zero
      --tls "ca-cert=/dgraph-tls/ca.crt; server-cert=/dgraph-tls/node.crt; server-key=/dgraph-tls/node.key; internal-port=true; client-cert=/dgraph-tls/client.zero1.crt; client-key=/dgraph-tls/client.zero1.key"
```

You can then run Dgraph Live Loader using the following:

```sh
# Now, connect to server using mTLS (mutual TLS)
$ dgraph live
   --tls "ca-cert=./tls/ca.crt; client-cert=./tls/client.dgraphuser.crt; client-key=./tls/client.dgraphuser.key; server-name=localhost; internal-port=true" \
   -s 21million.schema \
   -f 21million.rdf.gz
```

### Client Authentication Options

The server will always **request** client authentication.  There are four
different values for the `client-auth-type` option that change the security
policy of the client certificate.

| Value              | Client Cert/Key | Client Certificate Verified |
|--------------------|-----------------|--------------------|
| `REQUEST`          | optional        | Client certificate is not VERIFIED if provided. (least secure) |
| `REQUIREANY`       | required        | Client certificate is never VERIFIED |
| `VERIFYIFGIVEN`    | optional        | Client certificate is VERIFIED if provided (default) |
| `REQUIREANDVERIFY` | required        | Client certificate is always VERIFIED (most secure) |

`REQUIREANDVERIFY` is the most secure but also the most difficult to configure
for clients. When using this value, the value of `server-name` is matched
against the certificate SANs values and the connection host.

{{% notice "note" %}}If mTLS is enabled using `internal-port=true`,
internal ports (by default, 5080 and 7080) use the `REQUIREANDVERIFY` setting.
Unless otherwise configured, external ports (by default, 9080, 8080 and 6080)
use the `VERIFYIFGIVEN` setting. Changing the `client-auth-type` option to
another setting only affects client authentication on external ports.{{% /notice %}}

## Using Ratel UI with Client authentication

Ratel UI (and any other JavaScript clients built on top of `dgraph-js-http`)
connect to Dgraph servers via HTTP, when TLS is enabled servers begin to expect
HTTPS requests only.

If you haven't already created the CA certificate and the node certificate for alpha servers from the earlier instructions (see [Dgraph Certificate Management Tool](#dgraph-certificate-management-tool)), the first step would be to generate these certificates, it can be done by the following command:
```sh
# Create rootCA and node certificates/keys
$ dgraph cert -n localhost
```

If Dgraph Alpha's `client-auth-type` option is set to `REQUEST` or `VERIFYIFGIVEN`
(default), then client certificate is not mandatory. The steps after generating
CA/node certificate are as follows:

### Step 1. Install Dgraph Root CA into System CA
##### Linux (Debian/Ubuntu)
```sh
# Copy the generated CA to the ca-certificates directory
$ cp /path/to/ca.crt /usr/local/share/ca-certificates/ca.crt
# Update the CA store
$ sudo update-ca-certificates`
```

### Step 2. Install Dgraph Root CA into Web Browsers Trusted CA List

##### Firefox

* Choose Preferences -> Privacy & Security -> View Certificates -> Authorities
* Click on Import and import the `ca.crt`

##### Chrome

* Choose Settings -> Privacy and Security -> Security -> Manage Certificates -> Authorities
* Click on Import and import the `ca.crt`

### Step 3. Point Ratel to the `https://` endpoint of alpha server.

* Change the Dgraph Alpha server address to `https://` instead of `http://`, for example `https://localhost:8080`.

For `REQUIREANY` and `REQUIREANDVERIFY` as `client-auth-type` option, you need to follow the steps above and you
also need to install client certificate on your browser:

1. Generate a client certificate: `dgraph cert -c laptopuser`.
2. Convert it to a `.p12` file:
   ```sh
   openssl pkcs12 -export \
      -out laptopuser.p12 \
      -in tls/client.laptopuser.crt \
      -inkey tls/client.laptopuser.key
   ```
   Use any password you like for export, it is used to encrypt the p12 file.

3. Import the client certificate to your browser. It can be done in Chrome as follows:
   * Choose Settings -> Privacy and Security -> Security -> Manage Certificates -> Your Certificates
   * Click on Import and import the `laptopuser.p12`.

{{% notice "note" %}}
Mutual TLS may not work in Firefox because Firefox is unable to send privately-signed client certificates, this issue is filed [here](https://bugzilla.mozilla.org/show_bug.cgi?id=1662607).
{{% /notice %}}


Next time you use Ratel to connect to an alpha with Client authentication
enabled the browser will prompt you for a client certificate to use. Select the client's
certificate you've imported in the step above and queries/mutations will
succeed.

## Using Curl with Client authentication

When TLS is enabled, `curl` requests to Dgraph will need some specific options to work.  For instance (for an export request):

```
curl --silent https://localhost:8080/admin/export
```

If you are using `curl` with [Client Authentication](#client-authentication-options) set to `REQUIREANY` or `REQUIREANDVERIFY`, you will need to provide the client certificate and private key.  For instance (for an export request):

```
curl --silent --cacert ./tls/ca.crt --cert ./tls/client.dgraphuser.crt --key ./tls/client.dgraphuser.key https://localhost:8080/admin/export
```

Refer to the `curl` documentation for further information on its TLS options.

## Access Data Using a Client

Some examples of connecting via a [Client](/clients) when TLS is in use can be found below:

- [dgraph4j](https://github.com/dgraph-io/dgraph4j#creating-a-secure-client-using-tls)
- [dgraph-js](https://github.com/dgraph-io/dgraph-js/tree/master/examples/tls)
- [dgo](https://github.com/dgraph-io/dgraph/blob/main/tlstest/acl/acl_over_tls_test.go)
- [pydgraph](https://github.com/dgraph-io/pydgraph/tree/master/examples/tls)

## Troubleshooting Ratel's Client authentication

If you are getting errors in Ratel when TLS is enabled, try opening your Dgraph
Alpha URL as a web page.

Assuming you are running Dgraph on your local machine, opening
`https://localhost:8080/` in the browser should produce a message `Dgraph browser is available for running separately using the dgraph-ratel binary`.

In case you are getting a connection error, try not passing the
`client-auth-type` flag when starting an alpha. If you are still getting an
error, check that your hostname is correct and the port is open; then make sure
that "Dgraph Root CA" certificate is installed and trusted correctly.

After that, if things work without passing `client-auth-type` but stop working when
`REQUIREANY` and `REQUIREANDVERIFY` are set, make sure the `.p12` file is
installed correctly.
