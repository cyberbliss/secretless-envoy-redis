# secretless-envoy-redis
PoC showing a secretless broker for Redis using Envoy and cert-manager.

## Pre-requisites

- A running and accessible K8s cluster, to which you have permissions to deploy resources (tested against Kind and GKE Standard)
- `helmfile` [installed](https://helmfile.readthedocs.io/en/latest/#installation)

## Install cert-manager + addons

cert-manager plus its addons are used to generate, sign, distribute and manage X509 SVIDs. These all need to be installed first and it is helpful to do it in a deterministic order, hence using Helmfile to manage this.

```bash
set -a; source .env; set +a; helmfile apply --file ./bootstrap-helmfile.yaml.gotmpl --interactive
```

After a few minutes the various deployments in the cert-manager namespace should be ready:

```
cert-manager                              1/1     1            1           2m59s
cert-manager-approver-policy              1/1     1            1           2m25s
cert-manager-cainjector                   1/1     1            1           2m59s
cert-manager-csi-driver-spiffe-approver   1/1     1            1           101s
cert-manager-webhook                      1/1     1            1           2m59s
trust-manager                             1/1     1            1           2m7s
```

## Install example-app (optional)

This example-app just deploys an Alpine openssl Pod with a csi-driver-spiffe volume mounted so that its X509 SVID can be accessed from the Pod's container.

```bash
set -a; source .env; set +a; helmfile apply --file ./example-app-helmfile.yaml.gotmpl --interactive
```

The example-app Pod is deployed into the `sandbox` namespace. Once it is running you can exec into it and view its X509 crt and key files (mounted in /var/run/secrets/spiffe.io).

```
/ # cd /var/run/secrets/spiffe.io
/var/run/secrets/spiffe.io # ls -l
total 0
lrwxrwxrwx    1 root     root            14 Apr 16 07:48 tls.crt -> ..data/tls.crt
lrwxrwxrwx    1 root     root            14 Apr 16 07:48 tls.key -> ..data/tls.key
/var/run/secrets/spiffe.io #
```

To view the cert:

```bash
openssl x509 -in /var/run/secrets/spiffe.io/tls.crt -text -noout
```

Notice that:

1. The issuer is `CN=csi-driver-spiffe-ca` - this is the self-signed Root CA that was generated as part of the cert-manager bootstrapping. 
2. The Spiffe ID is stored in the `Subject Alternative Name` field:

```
 X509v3 Subject Alternative Name: critical
                URI:spiffe://spire-cm-poc.example.com/ns/sandbox/sa/example-app
```

## Install web-app (optional)

The web-app application consists of a Backend Pod, which serves some REST endpoints providing bank account data, and 2 Frontend Pods which call the backend APIs and render the data into a web page. Each Frontend Pod displays a different account. The frontend and backend communicate via Envoy proxy sidecars; these sidecars establish a mTLS connection between frontend and backend using each Pod's SVID. Furthermore, the backend Pod's Envoy config is setup to validate which SVIDs are permitted to call the backend API endpoints, i.e. not only does the SVID need to be valid it also has to be explicitly permitted to call the APIs, thus providing both authn and authz. 

```bash
set -a; source .env; set +a; helmfile apply --file ./web-app-helmfile.yaml.gotmpl --interactive
```

To control which frontend Pods can call the Backend APIs change the values of `validSA` in `web-app-helmfile.yaml`:

```yaml
  values:
  - spiffeTrustDomain: {{ requiredEnv "SPIFFE_TRUST_DOMAIN" }}
  - validSA:
    - "frontend"
#   - "frontend-2" 
```

To verify it's all working as expected, port forward to each Frontend service and view in a browser, e.g.

```bash
kubectl -n default port-forward service/frontend 10000:80
```

If the backend API is successfully called then the web page will be populated with bank transactions.

## Install redis-app

The redis-app application consists of a Redis server Pod and 2 Redis client Pods. Each Pod also contains an Envoy proxy sidecar which is responsible for providing a mTLS connection between the client and server Pods, using X509 SVIDs provided by cert-manager. The default configuration allows connectivity from the `redis-client` Pod but not `redis-client2`. The purpose of this application setup is to demonstrate the following using SVIDs and Envoy:

1. mTLS between client and server without requiring special configuration changes for either the client or the server
2. Using the SVID to control which client is permitted to access the server
3. Using Envoy's Redis proxy to control which commands a client can successfully execute against the server, i.e. no admin related commands

```bash
set -a; source .env; set +a; helmfile apply --file ./redis-app-helmfile.yaml.gotmpl --interactive
```

Once all Pods are running exec into the `redis-client` Pod and use the `redis-cli` to connect: `redis-cli -h localhost -p 5000`; commands like `PING`, `SET` and `GET` will work correctly. Certain admin related commands will not work because not only is Envoy used to provide a mTLS connection and control which client Pod can connect, the Envoy sidecar in the server Pod also uses a [Redis proxy](https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/other_protocols/redis) which only supports a subset of Redis commands. 

To enable both client Pods to connect to the server edit `redis-app-helmfile.yaml.gotmpl` like so:

```yaml
  - validSA:
    - "redis-client"
    - "redis-client2" 
```

Once the changes have been applied the `redis-server` Pod will need to be restarted to pick up the changes.

