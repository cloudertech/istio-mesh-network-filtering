# Prerequisities
```
  Istio service mesh installed
  Kubernetes 1.17 (AWS EKS)
```

# Create a namespace (secpoc)
```
kubectl apply -f ns.yaml

---
apiVersion: v1
kind: Namespace
metadata:
  name: secpoc
  labels:
    istio-injection: enabled
    networking/namespace: secpoc

---
apiVersion: v1
kind: Namespace
metadata:
  name: altsecpoc
  labels:
    istio-injection: enabled
    networking/namespace: altsecpoc
```

# Create a test pod

```
kubectl apply -f sleep.yaml

export SOURCE_POD=$(kubectl get pod -l app=sleep -o jsonpath={.items..metadata.name})

kubectl exec -it "$SOURCE_POD" -n secpoc -c sleep -- id
uid=65534(nobody) gid=65533(nogroup) groups=1337


kubectl apply -f httpbin.yaml

```

# Create network policies

    [] Verify that pod can not connect the external endpoint  (Denied by REGISTRY_ONLY)
    [] Verify that pod can connect another pod in another namespaces
    [] Verify that pod can connect kubernetes api
    [] Verify that pod can connect to EC2 metadata
    [] create serviceentry
    [] Verify that pod can  access the external endpoint
    [] Default deny per namespace
    [] Allow access to istio-system, Core DNS
    [] Block access to kubernetes api
    [] Block access to EC2 metadata

    export SOURCE_POD=$(kubectl get pod -l app=sleep -o jsonpath={.items..metadata.name})
    kubectl exec "$SOURCE_POD" -n secpoc -c sleep -- curl -v -sL -o /dev/null -D - https://api.ipify.org
    
    *   Trying 54.225.169.28:443...
    * Connected to api.ipify.org (54.225.169.28) port 443 (#0)
    ....
    * OpenSSL SSL_connect: Connection reset by peer in connection to api.ipify.org:443 
    * Closing connection 0
    command terminated with exit code 35


    kubectl logs -f $SOURCE_POD -c istio-proxy --tail=10
    ... snip ...
    [2020-12-03T10:46:10.882Z] "- - -" 0 UH "-" "-" 0 0 0 - "-" "-" "-" "-" "-" - - 54.225.169.28:443 10.10.167.168:41034 - -

    kubectl exec "$SOURCE_POD" -n secpoc -c sleep -- curl -sL -o /dev/null -D - http://httpbin.altsecpoc:8000/status/200

    kubectl exec "$SOURCE_POD" -n secpoc -c sleep -- curl -v -sL -o /dev/null -D - https://kubernetes.default
     
    kubectl apply -f serviceentry.yaml 
    serviceentry.networking.istio.io/ipify created


    kubectl exec "$SOURCE_POD" -n secpoc -c sleep -- curl -v -sL -o /dev/null -D - https://api.ipify.org
    200 OK


    kubectl logs -f $SOURCE_POD -c istio-proxy --tail=10
    ... snip ...
    [2020-12-03T10:51:20.375Z] "- - -" 0 - "-" "-" 780 5162 368 - "-" "-" "-" "-" "54.235.182.194:443" outbound|443||api.ipify.org 10.10.167.168:60286 54.243.164.148:443 10.10.167.168:52330 api.ipify.org -


    kubectl apply -f network-policies.yaml


    $ kubectl exec "$SOURCE_POD" -n secpoc -c sleep -- curl -v -sL -o /dev/null -D - https://api.ipify.org
    *   Trying 54.243.164.148:443...
    * Connected to api.ipify.org (54.243.164.148) port 443 (#0)
    ....
    * OpenSSL SSL_connect: Connection reset by peer in connection to api.ipify.org:443 
    * Closing connection 0
    command terminated with exit code 35


    kubectl logs -f $SOURCE_POD -c istio-proxy --tail=10
    [2020-12-03T10:54:15.891Z] "- - -" 0 UF,URX "-" "-" 0 0 10009 - "-" "-" "-" "-" "23.21.42.25:443" outbound|443||api.ipify.org - 54.243.164.148:443 10.10.167.168:53850 - -


    kubectl exec "$SOURCE_POD" -n secpoc -c sleep -- curl -v -sL -o /dev/null -D - https://kubernetes.default
    
    kubectl logs -f $SOURCE_POD -c istio-proxy --tail=10
    2020-12-03T10:55:38.255Z] "- - -" 0 UF,URX "-" "-" 0 0 10013 - "-" "-" "-" "-" "10.10.144.80:443" outbound|443||kubernetes.default.svc.cluster.local - 172.20.0.1:443 10.10.167.168:39590

    kubectl exec "$SOURCE_POD" -n secpoc -c sleep -- curl -v -sL -o /dev/null -D - http://169.254.169.254
    
    kubectl logs -f $SOURCE_POD -c istio-proxy --tail=10

# Give access to an external endpoint through egress-gw
    [] make sure there's a serviceentry object
    [] create egress rules


    kubectl apply -f egress-gw-rules.yaml

    Add label  networking/allow-through-egress-gateway: "true"  to the sleep pod


    kubectl exec "$SOURCE_POD" -n secpoc -c sleep -- curl -sL -o /dev/null -D - https://api.ipify.org
    HTTP/1.1 200 OK
    Server: Cowboy
    Connection: keep-alive
    Content-Type: text/plain
    Vary: Origin
    Date: Thu, 03 Dec 2020 11:06:08 GMT
    Content-Length: 12
    Via: 1.1 vegur


    kubectl logs -f $SOURCE_POD -c istio-proxy --tail=10
    [2020-12-03T11:06:08.497Z] "- - -" 0 - "-" "-" 780 5162 359 - "-" "-" "-" "-" "10.10.154.22:8443" outbound|443|ipify|istio-egressgateway.istio-system.svc.cluster.local 10.10.167.168:41254 54.225.66.103:443 10.10.167.168:51564 api.ipify.org -

    stern -n istio-system istio-egressgateway --tail=10

    istio-egressgateway-64b4bcc48f-4zdjp istio-proxy [2020-12-03T11:06:08.499Z] "- - -" 0 - "-" "-" 780 5162 356 - "-" "-" "-" "-" "54.235.83.248:443" outbound|443||api.ipify.org 10.10.154.22:40086 10.10.154.22:8443 10.10.167.168:41254 api.ipify.org -




# These ones below are nice to haves but not mandatory for the sake of the traffic filtering demo

# Create namespace-wide and pod-level resource limits

```
kubectl apply -f resource-limits.yaml 
resourcequota/mem-cpu-quota configured
limitrange/default-limits configured
```

# Force mTLS within namespace
```
Show mtls status
```

# Create default PSP
```
Disallow Root user
Set running user/group
Disallow Capabilities etc.
```
