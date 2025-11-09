## Purpose of RequestAuthentication in Istio

A RequestAuthentication resource defines how Istio should validate JSON Web Tokens (JWTs) coming into workloads.
It doesn’t enforce access rules — it just tells Istio “if a request has a JWT, here’s how to verify it.”


Typical use:

- Microservices that require JWT-based authentication.

- Gateways or workloads needing to validate tokens issued by an external IdP (like Auth0, Cognito, Okta, etc.)

### 🧱 Template Logic Explanation

1. {{- range $key, $val := .Values.requestAuthentication.services }}

Loops over each service entry defined in your values file.

So if your values.yaml has:

```yaml
requestAuthentication:
  services:
    - name: api-auth
      namespace: platform
      selector: my-api
      jwkRules:
        - issuer: "https://auth.example.com/"
          jwksUri: "https://auth.example.com/.well-known/jwks.json"
```

2. metadata

```yaml
metadata:
  name: {{ $val.name }}
  namespace: {{ $val.namespace }}
```
3. Selector (optional)

```yaml
selector:
  matchLabels:
    app.kubernetes.io/name: {{ $val.selector }}
```

If provided, limits JWT validation only to pods matching this label (usually per workload).
If omitted → applies cluster-wide in that namespace.

4. JWT Rules

```yaml
jwtRules:
  - issuer: {{ $jwks.issuer }}
    jwksUri: {{ $jwks.jwksUri }}
```
Defines where Istio should fetch public keys to validate tokens.
These correspond to the OpenID Connect discovery endpoints of your IdP.

5. Headers and Prefix

```yaml
fromHeaders:
- name: {{ $jwks.jwtHeaders }}
  prefix: "Bearer "

Tells Istio where to extract the JWT from — typically:

yaml
Copy code
jwtHeaders: Authorization
```

6. Token Forwarding

```yaml
forwardOriginalToken: true
```

Ensures that the original token is passed along to backend services after Istio validates it.
This allows downstream services to also use the token (e.g., for user info extraction).


## lets go through an example mate

You have a microservice named “payments” deployed in the namespace platform.
You want to secure it so that:

- All requests must include a valid JWT issued by Auth0.

- Istio validates the JWT (signature, expiry, issuer) before traffic reaches the app.

- Only authenticated requests are allowed (we’ll pair with AuthorizationPolicy later).


### 🧩 Step 1 – Helm values.yaml


```yaml
requestAuthentication:
  services:
    - name: payments-auth
      namespace: platform
      selector: payments
      jwkRules:
        - issuer: "https://dev-1234.auth0.com/"
          jwksUri: "https://dev-1234.auth0.com/.well-known/jwks.json"
          jwtHeaders: "Authorization"
          prefix: true
```

### 🧱 Step 2 – Rendered YAML from your template

```yaml
---
apiVersion: security.istio.io/v1beta1
kind: RequestAuthentication
metadata:
  name: payments-auth
  namespace: platform
spec:
  selector:
    matchLabels:
      app.kubernetes.io/name: payments
  jwtRules:
  - issuer: https://dev-1234.auth0.com/
    jwksUri: https://dev-1234.auth0.com/.well-known/jwks.json
    fromHeaders:
    - name: Authorization
      prefix: "Bearer "
    forwardOriginalToken: true
```

this tells Istio:

“For pods labeled app.kubernetes.io/name=payments in the platform namespace,
if a request has a JWT issued by https://dev-1234.auth0.com/,
validate it using the keys at the JWKS URI.”

### 🧠 Step 3 – Deploy the workload

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: payments
  namespace: platform
  labels:
    app.kubernetes.io/name: payments
spec:
  replicas: 1
  selector:
    matchLabels:
      app.kubernetes.io/name: payments
  template:
    metadata:
      labels:
        app.kubernetes.io/name: payments
    spec:
      containers:
        - name: payments
          image: myrepo/payments:latest
          ports:
            - containerPort: 8080
```

### 🔒 Step 4 – Add an AuthorizationPolicy

The RequestAuthentication only validates JWTs, but you also want to enforce that requests must have valid JWTs.

```yaml
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: payments-authz
  namespace: platform
spec:
  selector:
    matchLabels:
      app.kubernetes.io/name: payments
  action: ALLOW
  rules:
  - from:
      - source:
          requestPrincipals: ["*"]  # Allow any authenticated identity
```

This means:

Only requests with valid JWTs (matching the issuer above) are allowed.

Any request without a valid token is rejected with HTTP 401 Unauthorized.


