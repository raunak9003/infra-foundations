now in order to test side car injection i tried to curl the production service


```yaml
raunak_jhunjhunwala_9003@Raunaks-MacBook-Pro playground % kubectl get svc
NAME          TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)    AGE
details       ClusterIP   10.103.100.174   <none>        9080/TCP   94m
kubernetes    ClusterIP   10.96.0.1        <none>        443/TCP    158m
productpage   ClusterIP   10.96.169.215    <none>        9080/TCP   94m
ratings       ClusterIP   10.103.53.162    <none>        9080/TCP   94m
reviews       ClusterIP   10.96.33.27      <none>        9080/TCP   94m
raunak_jhunjhunwala_9003@Raunaks-MacBook-Pro playground % minikube ssh
docker@minikube:~$ curl 10.96.169.215:9080          

<meta charset="utf-8">
<meta http-equiv="X-UA-Compatible" content="IE=edge">
<meta name="viewport" content="width=device-width, initial-scale=1">
```
and shockingly i got to know that i was able to access the service despite of mtls.
but why??

its bcz initially the istion follows the perissive mtls which means that instead of having a valid certificate we can access the service.
below was the response on accessing the product page service

```yaml
<style>
    table {
        color: #333;
        background: white;
        border: 1px solid grey;
        font-size: 12pt;
        border-collapse: collapse;
        width: 100%;
    }

    table thead th,
    table tfoot th {
        color: #fff;
        background: #466BB0;
    }

    table caption {
        padding: .5em;
    }

    table th,
    table td {
        padding: .5em;
        border: 1px solid lightgrey;
    }
</style>



<script src="static/tailwind/tailwind.css"></script>



<div class="mx-auto px-4 sm:px-6 lg:px-8">
    <div class="flex flex-col space-y-5 py-32 mx-auto max-w-7xl">
        <h3 class="text-2xl">Hello! This is a simple bookstore application consisting of three services as shown below
        </h3>
        
        <table class="table table-condensed table-bordered table-hover"><tr><th>name</th><td>http://details:9080</td></tr><tr><th>endpoint</th><td>details</td></tr><tr><th>children</th><td><table class="table table-condensed table-bordered table-hover"><thead><tr><th>name</th><th>endpoint</th><th>children</th></tr></thead><tbody><tr><td>http://details:9080</td><td>details</td><td></td></tr><tr><td>http://reviews:9080</td><td>reviews</td><td><table class="table table-condensed table-bordered table-hover"><thead><tr><th>name</th><th>endpoint</th><th>children</th></tr></thead><tbody><tr><td>http://ratings:9080</td><td>ratings</td><td></td></tr></tbody></table></td></tr></tbody></table></td></tr></table>
        
        <p>
            Click on one of the links below to auto generate a request to the backend as a real user or a tester
        </p>
        <ul>
            <li>
                <a href="/productpage?u=normal" class="text-blue-500 hover:text-blue-600">Normal user</a>
            </li>
            <li>
                <a href="/productpage?u=test" class="text-blue-500 hover:text-blue-600">Test user</a>
            </li>
        </ul>
    </div>
</div>
docker@minikube:~$ 
```


then i tried to implement the proper strict mtls and then i wasnot able to access the micro-service.


```yaml
docker@minikube:~$ curl 10.96.169.215:9080
curl: (56) Recv failure: Connection reset by peer
docker@minikube:~$ 
```
