## 📘 Why Reflector Cannot Handle Certificate Merging (and Why Trust Manager Can)

### 🪶 What Reflector Does

Reflector is a lightweight Kubernetes controller that simply copies Secrets or ConfigMaps from one namespace to another. It doesn’t modify the data — it just replicates objects as they are.

### ⚙️ What Merging Certificates Means

Certificates often form a chain of trust:

```
Root CA  →  Intermediate CA  →  Application Certificate
```

To make sure every service trusts this chain, you need to combine (merge) the Root and Intermediate certificates into a single bundle file:

```
-----BEGIN CERTIFICATE-----
Root CA
-----END CERTIFICATE-----
-----BEGIN CERTIFICATE-----
Intermediate CA
-----END CERTIFICATE-----
```

If part of the chain is missing, you get errors like:
```
x509: certificate signed by unknown authority
```

### 🚫 Why Reflector Cannot Handle Merging

Reflector only copies resources — it cannot:

Open multiple Secrets

- Read or understand the certificate data inside

- Combine Root + Intermediate certificates in order

- Create a new “merged” ConfigMap or Secret


So Reflector can copy:

```
root-ca Secret ✅
intermediate-ca Secret ✅
```

But it cannot create:

```
merged-ca-bundle ❌
```


## summary

| Capability                              | Reflector |
| --------------------------------------- | --------- |
| Copy a Secret or ConfigMap              | ✅ Yes     |
| Watch and update on change              | ✅ Yes     |
| Read inside and understand certificates | ❌ No      |
| Merge Root + Intermediate + Partner CAs | ❌ No      |
| Auto rebuild when certs renew           | ❌ No      |
