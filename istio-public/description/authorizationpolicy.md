## 🧩 What is AuthorizationPolicy in Istio?

An AuthorizationPolicy decides who can access what inside your cluster.

- It runs inside the Envoy proxy (like istio-proxy) next to your app.

- It looks at incoming requests (after JWT is validated, if you have RequestAuthentication).

- It can check:

- the source of the request (who sent it),

- the destination (which path, port, or method is called),

- and optional conditions (claims, IPs, namespaces, etc.).

It then decides:
👉 ALLOW or DENY the request.


## Helm template — overview

```yaml
{{- range $key, $val := .Values.authorizationPolicy.services }}
---
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: {{ $val.name }}
  namespace: {{ $val.namespace }}
spec:
  ...
{{- end }}
```

So this loop will create one policy per service you define in values.yaml.

Now, inside spec, you’re building:

a selector — which pods it applies to,

an action — ALLOW or DENY,

and rules — what kind of requests are allowed or denied.

## rules

This is the heart ❤️ of AuthorizationPolicy.

Rules describe who → can do → what.

Your template covers three parts inside each rule:

```yaml
from:   # who can call
to:     # what they can access
when:   # extra conditions
```

### from – who is allowed

```
from:
- source:
    principals: ["cluster.local/ns/platform/sa/frontend"]
```

This means only requests coming from the frontend service account in the platform namespace are allowed.


| Field               | Meaning                                                      |
| ------------------- | ------------------------------------------------------------ |
| `principals`        | Authenticated identity (like service account or user in JWT) |
| `namespaces`        | Namespace of the source                                      |
| `ipBlocks`          | Client IPs                                                   |
| `requestPrincipals` | JWT identities (`issuer/subject`)                            |


```
to:
- operation:
    methods: ["GET", "POST"]
    paths: ["/api/v1/orders/*"]
    ports: ["8080"]
```


This defines what part of your app is allowed:

HTTP methods (GET, POST, DELETE)

URLs (paths)

ports

So you can say things like:

“Only GET requests to /customers are allowed.”


### when – extra conditions
```
when:
- key: request.auth.claims[role]
  values: ["admin"]
```

This checks claims inside the JWT — so you can enforce role-based rules.

✅ Example:
Only users whose token has "role": "admin" are allowed.



