# Protecting an API with Authorino on OpenShift

This guide exemplifies an application (_"Talker API"_) protected with [Envoy](https://www.envoyproxy.io/) and [Authorino](https://github.com/kuadrant/authorino), in an OpenShift cluster, using OpenShift OAuth, Kubernetes tokens and Keycloak for authentication.

**Stack:**
- [OpenShift](https://www.redhat.com/en/technologies/cloud-computing/openshift) cluster
- [Talker API](https://github.com/kuadrant/authorino-examples#talker-api) with an Envoy sidecar proxy
- [Authorino](https://github.com/kuadrant/authorino) authorization service
- [Keycloak](https://www.keycloak.org/) server (optional, for more advanced use cases)

**Requirements:**
- OpenShift cluster (admin privilege to the cluster needed for a few steps – e.g. install the Authorino Operator, define OAuth clients)
- OpenShift `oc` CLI tool
- [jq](https://stedolan.github.io/jq/)

**Outline:**<br/>
![Steps](http://www.plantuml.com/plantuml/png/RP3FJiCm3CRlUGfhTuRummLDqogGOEB03Xmh8I-rkgYfKoNkeI3U7QiB9OTTOd-sV_hix99WbB7tPfDayhGraQmWjvxWsm2yXkY-0WlwohkMUs81gmz5ysCsrvafeDNDkkPd6doOG4vKymVwZY9KX_qAC87CyXC7LqAt2kqvQTFNN8roKbiECu1_gfo_q_b33AALYox3kLSYzugy7mKT0tBDQ2qbNITqn8hah0GU57WAdCQUBdhOSwz4NeBZjkOZJO6RHxsaQU2D9ki3TZFJPM7C_p_0rROuSicql9oHevRoclEhSbaYHrXl2uyTSJFs_XS0)

## Before you begin

- Make sure you are logged in to you OpenShift cluster using the OpenShift `oc` CLI tool
- Export the termination domain of your OpenShift cluster to a shell variable (referred in the steps below):
  ```
  export OPENSHIFT_DOMAIN=<openshift domain here>
  ```

## 1. Deploy the Talker API

```sh
kubectl apply -f -<<EOF
apiVersion: v1
kind: Namespace
metadata:
  name: talker-api
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: envoy
  namespace: talker-api
  labels:
    app: envoy
data:
  envoy.yaml: |
    static_resources:
      clusters:
      - name: talker-api
        connect_timeout: 0.25s
        type: strict_dns
        lb_policy: round_robin
        load_assignment:
          cluster_name: talker-api
          endpoints:
          - lb_endpoints:
            - endpoint:
                address:
                  socket_address:
                    address: 127.0.0.1
                    port_value: 3000
      - name: authorino
        connect_timeout: 0.25s
        type: strict_dns
        lb_policy: round_robin
        http2_protocol_options: {}
        load_assignment:
          cluster_name: authorino
          endpoints:
          - lb_endpoints:
            - endpoint:
                address:
                  socket_address:
                    address: authorino-authorino-authorization
                    port_value: 50051
      listeners:
      - address:
          socket_address:
            address: 0.0.0.0
            port_value: 8000
        filter_chains:
        - filters:
          - name: envoy.http_connection_manager
            typed_config:
              "@type": type.googleapis.com/envoy.extensions.filters.network.http_connection_manager.v3.HttpConnectionManager
              stat_prefix: local
              route_config:
                name: talker_api
                virtual_hosts:
                - name: talker_api
                  domains: ['*']
                  routes:
                  - match: { prefix: / }
                    route:
                      cluster: talker-api
              http_filters:
              - name: envoy.filters.http.ext_authz
                typed_config:
                  "@type": type.googleapis.com/envoy.extensions.filters.http.ext_authz.v3.ExtAuthz
                  transport_api_version: V3
                  failure_mode_allow: false
                  include_peer_certificate: true
                  grpc_service:
                    envoy_grpc:
                      cluster_name: authorino
                    timeout: 1s
              - name: envoy.filters.http.router
                typed_config: {}
              use_remote_address: true
    admin:
      access_log_path: "/tmp/admin_access.log"
      address:
        socket_address:
          address: 0.0.0.0
          port_value: 8001
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: talker-api
  namespace: talker-api
  labels:
    app: talker-api
spec:
  selector:
    matchLabels:
      app: talker-api
  template:
    metadata:
      labels:
        app: talker-api
    spec:
      containers:
      - name: talker-api
        image: quay.io/kuadrant/authorino-examples:talker-api
        imagePullPolicy: Always
        env:
        - name: PORT
          value: "3000"
        ports:
        - containerPort: 3000
        tty: true
      - name: envoy
        image: envoyproxy/envoy:v1.19-latest
        command:
        - /usr/local/bin/envoy
        args:
        - --config-path /usr/local/etc/envoy/envoy.yaml
        - --service-cluster front-proxy
        - --log-level info
        - --component-log-level filter:trace,http:debug,router:debug
        ports:
        - containerPort: 8000
          name: web
        - containerPort: 8001
          name: admin
        volumeMounts:
        - mountPath: /usr/local/etc/envoy
          name: config
          readOnly: true
      volumes:
      - configMap:
          items:
          - key: envoy.yaml
            path: envoy.yaml
          name: envoy
        name: config
  replicas: 1
---
apiVersion: v1
kind: Service
metadata:
  name: talker-api
  namespace: talker-api
  labels:
    app: talker-api
spec:
  ports:
  - name: envoy
    port: 8000
    protocol: TCP
  selector:
    app: talker-api
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: talker-api
  labels:
    app: talker-api
spec:
  rules:
  - host: talker-api.apps.dev-eng-ocp4.dev.3sca.net
    http:
      paths:
      - backend:
          service:
            name: talker-api
            port:
              number: 8000
        path: /
        pathType: Prefix
EOF
```

## 2. Install Authorino

Install the Authorino Operator (uses [OLM](https://olm.operatorframework.io/); requires admin to the cluster):

```sh
kubectl apply -f -<<EOF
apiVersion: v1
kind: Namespace
metadata:
  name: authorino-operator
---
apiVersion: operators.coreos.com/v1alpha1
kind: CatalogSource
metadata:
  name: operatorhubio-catalog
  namespace: authorino-operator
spec:
  sourceType: grpc
  image: quay.io/kuadrant/authorino-operator-catalog:latest
  displayName: Authorino Operator
---
apiVersion: operators.coreos.com/v1
kind: OperatorGroup
metadata:
  name: authorino-operator
spec: {}
---
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: authorino-operator
spec:
  channel: alpha
  installPlanApproval: Automatic
  name: authorino-operator
  source: operatorhubio-catalog
  sourceNamespace: authorino-operator
EOF
```

Request an Authorino instance:

```sh
kubectl -n talker-api apply -f -<<EOF
apiVersion: operator.authorino.kuadrant.io/v1beta1
kind: Authorino
metadata:
  name: authorino
spec:
  listener:
    tls:
      enabled: false
  oidcServer:
    tls:
      enabled: false
EOF
```

> **Note:** The command above will request an Authorino instance to be created and managed by the Authorino Operator, in the same namespace as the Talker API. This instance of Authorino will only watch resources defined in that namespace. For shared instances of Authorino across Kubernetes namespaces, check out the instructions to deploy Auhtorino in [`cluster-wide`](https://github.com/Kuadrant/authorino/blob/main/docs/architecture.md#cluster-wide-vs-namespaced-instances) reconciliation mode.

## 3. Protect the Talker API

Create the AuthConfig:

```sh
kubectl -n talker-api apply -f -<<EOF
apiVersion: authorino.kuadrant.io/v1beta1
kind: AuthConfig
metadata:
  name: talker-api
spec:
  hosts:
  - talker-api.apps.$OPENSHIFT_DOMAIN
  - talker-api.talker-api.svc.cluster
  identity:
  - name: openshift-users
    kubernetes:
      audiences: ["https://kubernetes.default.svc"]
  authorization:
  - name: check-openshift-group
    json:
      rules:
      - selector: auth.identity.groups
        operator: incl
        value: talker-api
  response:
  - name: x-auth-data
    json:
      properties:
      - name: username
        valueFrom: { authJSON: auth.identity.username }
EOF
```

Create an OpenShift group that users and service accounts will be required to be member of, to consume the Talker API:

```sh
kubectl -n talker-api apply -f -<<EOF
apiVersion: user.openshift.io/v1
kind: Group
metadata:
  name: talker-api
users: # the names below are just an example; edit the list as needed for your use case
- john
- jane
EOF
```

Authorino will accept all Kubernetes or OpenShift tokens that include the `"https://kubernetes.default.svc"` audience (`aud` claim), issued to a user (`sub` claim) who is member of a group `"talker-api"` (`groups` claim) in the OpenShift User Management system.

## 4. Obtain an access token from OpenShift

There are multiple ways to obtain a token with an OpenShift OAuth server:
1. Using OpenShift built-in OAuth server and implementing one of the supported OAuth2 flows;
2. As a real user, reusing the token issued by OpenShift as part of the authentication with the OpenShift CLI or OpenShift Console;
3. Via Kubernetes `TokenRequest` API, to issue a token associated with a Service Account

Whichever method you choose to obtain the token, all tokens will work with [Authorino Kubernetes TokenReview](https://github.com/Kuadrant/authorino/blob/main/docs/features.md#kubernetes-tokenreview-identitykubernetes) identity verification method.

Make sure to store the access token in a shell variable named `ACCESS_TOKEN` before moving to the next step, _Consuming the Talker API_.

### Obtaining an access token via OAuth flow

Generate a secret for the OAuth client:

```sh
CLIENT_SECRET=$(openssl rand -base64 32)
```

Define an OAuth client n OpenShift:

```sh
kubectl apply -f -<<EOF
kind: OAuthClient
apiVersion: oauth.openshift.io/v1
metadata:
 name: talker-api
secret: $CLIENT_SECRET
redirectURIs:
- "http://echo-api.3scale.net"
grantMethod: prompt
EOF
```

Check the OAuth configuration to confirm the authorization endpoint and the token endpoint:

```sh
kubectl -n default run oidc-config --attach --rm --restart=Never -q --image=curlimages/curl -- https://openshift.default.svc/.well-known/oauth-authorization-server -sS -k
```

Generate a random 'state' parameter for the next couple steps:

```sh
STATE=$(openssl rand -hex 4)
```

To obtain an authorization code, open the following URL (authorization endpoint) in a web browser of your preference:

```
https://oauth-openshift.apps.$OPENSHIFT_DOMAIN/oauth/authorize?client_id=talker-api&response_type=code&state=${STATE}&redirect_uri=http%3A%2F%2Fecho-api.3scale.net&scope=user:full
```

You might be requested to authenticate to OpenShift.

_Important!_ After redirect to the Echo API, note the 'code' parameter in the reponse.

Exchange the authorization code for an access token (token endpoint):

```sh
ACCESS_TOKEN=$(curl -d 'client_id=talker-api' -d "client_secret=$CLIENT_SECRET" -d 'grant_type=authorization_code' -d "code=<authorization code from the previous step>" -d "state=$STATE" -d 'redirect_uri=http%3A%2F%2Fecho-api.3scale.net' "https://oauth-openshift.apps.$OPENSHIFT_DOMAIN/oauth/token") | jq -r .access_token
```

### Obtaining an access token via OpenShift CLI

Provided you are authenticated in the OpenShift CLI, run the following command to see the token:

```sh
oc whoami -t
```

### Obtaining an access token via OpenShift Console

Logged in to the OpenShift Console (either having used your username and password or via SSO), you have two options to obtain a token:
- OpenShift Console session token cookie
- New login command for the OpenShift CLI from the OpenShift Console

#### Session cookie

Logged in to the OpenShift Console (either having used your username and password or via SSO), use the browser Developer Tools to inspect the stored cookies or network traffic. Lookup for a cookie with ID `openshift-session-token`. The value of the cookie is a valid OpenShift access token.

#### New login command

In the OpenShift Console,
1. Click on your name on the top right corner
2. Choose "Copy login command"
3. If requested, re-authenticate to OpenShift
4. Click on "Display token" to see the login command for the OpenShift CLI which includes the access token

### Obtaining an access token via Kubernetes `TokenRequest` API

You can request a token associated with a Kubernetes Service Account using the Kubernetes `TokenRequest` API. This method might not suit you if want to issue tokens for real users, unless you choose to represent the users as a `ServiceAccount` resources instead of as an OpenShift `User` resources. However, it is a good approach for issuing tokens to applications running inside or outside the cluster.

Kubernetes tokens work for OpenShift as well. The API will give you a token that is not an OpenShift opaque token, but actually a Kubernetes JWT, e.g.:

```sh
kubectl create --raw /api/v1/namespaces/<client-app-namespace>/serviceaccounts/<client-app-sa-name>/token -f -<<EOF | jq -r .status.token
{ "apiVersion": "authentication.k8s.io/v1", "kind": "TokenRequest", "spec": { "audiences": ["https://kubernetes.default.svc"], "expirationSeconds": 600 } }
EOF
```

### Reviewing a token

After obtaining a token by any of the methods above, you can inspect the token in the terminal by running the following command:

```sh
oc create -o yaml -f -<<EOF
apiVersion: authentication.k8s.io/v1
kind: TokenReview
spec:
  token: $ACCESS_TOKEN
EOF
```

## 5. Consume the Talker API

```sh
curl -H "Authorization: Bearer $ACCESS_TOKEN" http://talker-api.apps.$OPENSHIFT_DOMAIN
```

<br/>

## Going beyond: Keycloak and OpenShift

### Install Keycloak

```sh
oc process -f https://raw.githubusercontent.com/keycloak/keycloak-quickstarts/latest/openshift-examples/keycloak.yaml \
    -p KEYCLOAK_ADMIN=admin \
    -p NAMESPACE=keycloak \
| oc create -f -
```

```sh
KEYCLOAK_URL=https://$(oc get route keycloak --template='{{ .spec.host }}') &&
echo "" &&
echo "Keycloak:                 $KEYCLOAK_URL" &&
echo "Keycloak Admin Console:   $KEYCLOAK_URL/admin" &&
echo "Keycloak Account Console: $KEYCLOAK_URL/realms/<realm-name>/account" &&
echo ""
```

### Setting up Keycloak identity provider to authenticate to OpenShift

Create an OAuth client in Keycloak:

**Client ID:** `openshift`<br/>
**Client secret:** PRyyRExjWR6iEKvRoIrpa3eNMOwVGH4GePVEefgDu1E

Store the client secret in OpenShift:

```sh
oc create secret generic keycloak-idp --from-literal=clientSecret=PRyyRExjWR6iEKvRoIrpa3eNMOwVGH4GePVEefgDu1E -n openshift-config
```

Edit the `cluster` OAuth configuration to include the Keycloak identity provider:

```yaml
apiVersion: config.openshift.io/v1
kind: OAuth
metadata:
  name: cluster
spec:
  identityProviders:
  - name: keycloak
    mappingMethod: claim
    type: OpenID
    openID:
      clientID: openshift
      clientSecret:
        name: keycloak-idp
      extraScopes:
      - email
      - profile
      extraAuthorizeParameters:
        include_granted_scopes: "true"
      claims:
        preferredUsername:
        - preferred_username
        - email
        name:
        - given_name
        - name
        email:
        - email
        groups:
        - groups
      issuer: $KEYCLOAK_URL/realms/<realm-name>
```

### Setting up OpenShift identity provider to authenticate to Keycloak

Create a Service Account to work as a simple OAuth client:

```sh
kubectl -n keycloak apply -f -<<EOF
apiVersion: v1
kind: ServiceAccount
metadata:
  name: keycloak
  annotations:
    serviceaccounts.openshift.io/oauth-redirectreference.keycloak: >-
      {"kind":"OAuthRedirectReference","apiVersion":"v1","reference":{"kind":"Route","name":"keycloak"}}
EOF
```

Note the Service Account token:

```sh
oc sa get-token keycloak
```

Create the Identity Provider in Keycloak:

1. Log in to Keycloak Admin console (username: 'admin' / password: see environment variables of the `keycloak` `DeploymentConfig`)
2. Click on "Identity Providers"
3. From the "Add provider" list, select "Openshift v4"
4. Fill in the client ID and client secret

    **Client ID:** system:serviceaccount:keycloak:keycloak<br/>
    **Client Secret:** \<service-account-token>
