now what is virtual service??

instead of focussing on theory let me explain by an example
in simple words lets say this book application here we have 3 versions for rating service(one with no stars, one with red stars and one ith black stars)

now through istio we can have canary deployment that means when some one hits the ratings service then traffic is not sent to any one version only we can distribute it so that in future if there is any fault in new version then the old version is still receiving traffic which is important bcz it woulnot break the system if there is any error in new version.


now what is the use of virtual service.

lets say i have 3 versions for a service and i want to send tarffic to all 3 with specofied weighatage for each like 1-> 50, 2->30,3->20 then through virtual service i can notify that for product i 

1,2,3
12,13,23
123

now destination rule

A DestinationRule defines policies and subsets for a particular service after routing.
You can think of it as a map that tells Istio:

“Hey, the service ratings has these versions — v1, v2, and v3.
Each version can have its own settings like load balancing, connection pool, TLS, etc.”

🧠 Simple analogy

If VirtualService is the traffic controller 🚦
then DestinationRule is the legend / reference list 📘 that explains what each destination (v1, v2, v3) actually means.

🧱 Example scenario

Let’s take the same Bookinfo example again — 3 versions of ratings service:

Version	Label
ratings-v1	version: v1
ratings-v2	version: v2
ratings-v3	version: v3
🧾 DestinationRule for ratings service:
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: ratings-dest-rule
spec:
  host: ratings  # applies to the "ratings" service
  subsets:
    - name: v1
      labels:
        version: v1
    - name: v2
      labels:
        version: v2
    - name: v3
      labels:
        version: v3


This tells Istio:

There’s one service called ratings

It has 3 subsets (versions), each identified by a label on the Pod

```yaml
apiVersion: networking.istio.io/v1
kind: DestinationRule
metadata:
  annotations:
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"networking.istio.io/v1","kind":"DestinationRule","metadata":{"annotations":{},"name":"reviews","namespace":"default"},"spec":{"host":"reviews","subsets":[{"labels":{"version":"v1"},"name":"v1"},{"labels":{"version":"v2"},"name":"v2"},{"labels":{"version":"v3"},"name":"v3"}]}}
  creationTimestamp: "2025-10-11T09:33:12Z"
  generation: 1
  name: reviews
  namespace: default
  resourceVersion: "15676"
  uid: 0cddbb4f-896d-4e0f-8cc5-86cb04b64c29
spec:
  host: reviews
  subsets:
  - labels:
      version: v1
    name: v1
  - labels:
      version: v2
    name: v2
  - labels:
      version: v3
    name: v3
~               
```
