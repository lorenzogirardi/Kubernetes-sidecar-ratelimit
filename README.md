# Kubernetes-sidecar-ratelimit

For some reasons that i will share later on in a dedicated discussion, i had the need to create a rate limit inside the application.   
<br>

For this scenario we have 2 greddy options:  
- have the logic inside the app , in a *code* way  
- have a sidecar container that has this role
<br>
<br>

## the code way  

This is the simplest one but has some side effects,  
first of all can be reaused just for the same language apps.  
Same as before , having this rate inside the app , we are creating a role inside this app  
that can be good or bad depending of the strategy applied,  
imagine the request interceptor for example will consume cpu and may *false* metrics if we are not aware of it.  
<br>
But anyway , it can be a solution , so in a simple python code you have to add the following for example  
(i will use the usual [app](https://github.com/lorenzogirardi/py-test-backend/tree/ratelimit/docker) i'm using for labs stuff)
<br>
I'ts Flask so you just have to add the following:  
in requirements.txt  --> ```Flask-Limiter```  

Inside your code , import the libraries  
```
from flask_limiter import Limiter
from flask_limiter.util import get_remote_address
```

Enable the *limiter*  
```
limiter = Limiter(
	get_remote_address,
	app=app,
	storage_uri="memory://",
)
```  

And add the limiting option in the route u'd like to rate  
```
@limiter.limit("28 per second")
```
<br>

### Handson  
Just moved from 
```@limiter.limit("28 per second")``` to ```@limiter.limit("2 per second")```   

```
$ docker build -t lgirardi/pytbakrated .
[+] Building 11.1s (9/9) FINISHED
 => [internal] load build definition from Dockerfile                                                                                                                                                                                     0.0s
 => => transferring dockerfile: 34B                                                                                                                                                                                                      0.0s
 => [internal] load .dockerignore                                                                                                                                                                                                        0.0s
 => => transferring context: 2B                                                                                                                                                                                                          0.0s
 => [internal] load metadata for docker.io/library/python:3.9-alpine                                                                                                                                                                     0.6s
 => [internal] load build context                                                                                                                                                                                                        0.0s
 => => transferring context: 4.47kB                                                                                                                                                                                                      0.0s
 => CACHED [1/4] FROM docker.io/library/python:3.9-alpine@sha256:8bda1e9a98fa4e87ff6e3a7682f496532b06fcbae10326a59c8656126051d4df                                                                                                        0.0s
 => [2/4] COPY . /app                                                                                                                                                                                                                    0.0s
 => [3/4] WORKDIR /app                                                                                                                                                                                                                   0.0s
 => [4/4] RUN pip install -r requirements.txt                                                                                                                                                                                            9.9s
 => exporting to image                                                                                                                                                                                                                   0.4s
 => => exporting layers                                                                                                                                                                                                                  0.4s
 => => writing image sha256:583ec19905662414ad6bdb3b0da1042ec799ec46f8a676fac6553364cd02bf1e                                                                                                                                             0.0s
 => => naming to docker.io/lgirardi/pytbakrated
 ```   

```
$ docker run -p 5000:5000 lgirardi/pytbakrated
 * Serving Flask app 'app'
 * Debug mode: off
2023-02-11 10:36:06,264 INFO werkzeug MainThread : WARNING: This is a development server. Do not use it in a production deployment. Use a production WSGI server instead.
 * Running on all addresses (0.0.0.0)
 * Running on http://127.0.0.1:5000
 * Running on http://172.17.0.2:5000
2023-02-11 10:36:06,264 INFO werkzeug MainThread : Press CTRL+C to quit
```   


With a simple curl you can check if it's working , so the rate will be triggered if u go over 2 req/sec   
```
$ while true; do curl -I localhost:5000/api/fib/1 && sleep 0.4;done
HTTP/1.1 200 OK
Server: Werkzeug/2.2.2 Python/3.9.16
Date: Sat, 11 Feb 2023 10:37:54 GMT
Content-Type: text/html; charset=utf-8
Content-Length: 1
Vary: Accept-Encoding
Connection: close

HTTP/1.1 200 OK
Server: Werkzeug/2.2.2 Python/3.9.16
Date: Sat, 11 Feb 2023 10:37:54 GMT
Content-Type: text/html; charset=utf-8
Content-Length: 1
Vary: Accept-Encoding
Connection: close

HTTP/1.1 429 TOO MANY REQUESTS
Server: Werkzeug/2.2.2 Python/3.9.16
Date: Sat, 11 Feb 2023 10:37:55 GMT
Content-Type: text/html; charset=utf-8
Content-Length: 117
Vary: Accept-Encoding
Connection: close

HTTP/1.1 200 OK
Server: Werkzeug/2.2.2 Python/3.9.16
Date: Sat, 11 Feb 2023 10:37:55 GMT
Content-Type: text/html; charset=utf-8
Content-Length: 1
Vary: Accept-Encoding
Connection: close

HTTP/1.1 200 OK
Server: Werkzeug/2.2.2 Python/3.9.16
Date: Sat, 11 Feb 2023 10:37:56 GMT
Content-Type: text/html; charset=utf-8
Content-Length: 1
Vary: Accept-Encoding
Connection: close

HTTP/1.1 429 TOO MANY REQUESTS
Server: Werkzeug/2.2.2 Python/3.9.16
Date: Sat, 11 Feb 2023 10:37:56 GMT
Content-Type: text/html; charset=utf-8
Content-Length: 117
```  

in the same way the contrainer logs are showing the same  
```
2023-02-11 10:37:54,359 INFO werkzeug Thread-34 : 172.17.0.1 - - [11/Feb/2023 10:37:54] "HEAD /api/fib/1 HTTP/1.1" 200 -
2023-02-11 10:37:54,778 INFO werkzeug Thread-36 : 172.17.0.1 - - [11/Feb/2023 10:37:54] "HEAD /api/fib/1 HTTP/1.1" 200 -
2023-02-11 10:37:55,196 INFO flask-limiter Thread-38 : ratelimit 2 per 1 second (172.17.0.1) exceeded at endpoint: fib
2023-02-11 10:37:55,197 INFO werkzeug Thread-38 : 172.17.0.1 - - [11/Feb/2023 10:37:55] "HEAD /api/fib/1 HTTP/1.1" 429 -
2023-02-11 10:37:55,616 INFO werkzeug Thread-40 : 172.17.0.1 - - [11/Feb/2023 10:37:55] "HEAD /api/fib/1 HTTP/1.1" 200 -
2023-02-11 10:37:56,035 INFO werkzeug Thread-42 : 172.17.0.1 - - [11/Feb/2023 10:37:56] "HEAD /api/fib/1 HTTP/1.1" 200 -
2023-02-11 10:37:56,455 INFO flask-limiter Thread-44 : ratelimit 2 per 1 second (172.17.0.1) exceeded at endpoint: fib
2023-02-11 10:37:56,456 INFO werkzeug Thread-44 : 172.17.0.1 - - [11/Feb/2023 10:37:56] "HEAD /api/fib/1 HTTP/1.1" 429 -
```

Honestly there are few pro and cons ... but i'd like to enter in those details with a more practical exmaple ...  
just few concepts anyway are:
- is using the same pool of connections
- it's impacting the same metric that i'm using for autoscaling
- is stealing resources from app 
- is really simple rate limit that can fight with others stuff
<br>
<br>



## the sidecar way  

Yes i know , this is a *stretch*,  since we have service mesh , gateway etc etc that embrace this concept.  
However, for a specific reason, i created just a sidecar container that can be reused without the needs to implement the infrastructure mentioned before    

First of all the choice,
tons of possibility, from nginx reverse proxy , to haproxy to *whatever* proxy ...   
in this scenario i took the opportunity to use **envoy** , i've already used it with istio , and is really really flexible  
(sometimes too much, in a way to be confused :-)   


Anyway ... it's just a sidecar  
In an existing app you have to add the following to add envoy  
```
      - name: sidecar
        image: envoyproxy/envoy:v1.22-latest
        resources:
          limits:
            cpu: 100m
            memory: 150Mi
          requests:
            cpu: 30m
            memory: 55Mi
        ports:
          - name: http
            containerPort: 5002
            protocol: TCP
        volumeMounts:
          - name: sidecar-config
            mountPath: "/etc/envoy"
            readOnly: true
      volumes:
        - name: sidecar-config
          configMap:
            name: pytbakt-configmap
```
(great fantasy for the name :-)  


and ... well the most important configuration is the ```pytbakt-configmap``` that is the envoy configuration 
```
apiVersion: v1
kind: ConfigMap
metadata:
  name: pytbakt-configmap
  labels:
    app: pytbak
  namespace: pytbak
data:
  envoy.yaml: |
    static_resources:
      listeners:
      - name: listener_0
        address:
          socket_address:
            address: 0.0.0.0
            port_value: 5002
        filter_chains:
        - filters:
          - name: envoy.filters.network.http_connection_manager
            typed_config:
              "@type": type.googleapis.com/envoy.extensions.filters.network.http_connection_manager.v3.HttpConnectionManager
              stat_prefix: ingress_http
              route_config:
                name: local_route
                virtual_hosts:
                - name: local_service
                  domains: ["*"]
                  routes:
                  - match:
                      prefix: "/"
                    route:
                      cluster: pytbak
              http_filters:
              - name: envoy.filters.http.local_ratelimit
                typed_config:
                  "@type": type.googleapis.com/envoy.extensions.filters.http.local_ratelimit.v3.LocalRateLimit
                  stat_prefix: http_local_rate_limiter
                  token_bucket:
                    max_tokens: 29
                    tokens_per_fill: 29
                    fill_interval: 1s
                  filter_enabled:
                    runtime_key: local_rate_limit_enabled
                    default_value:
                      numerator: 100
                      denominator: HUNDRED
                  filter_enforced:
                    runtime_key: local_rate_limit_enforced
                    default_value:
                      numerator: 100
                      denominator: HUNDRED
                  response_headers_to_add:
                  - append_action: OVERWRITE_IF_EXISTS_OR_ADD
                    header:
                      key: x-local-rate-limit
                      value: 'true'
                  local_rate_limit_per_downstream_connection: false

              - name: envoy.filters.http.router
                typed_config:
                  "@type": type.googleapis.com/envoy.extensions.filters.http.router.v3.Router

      clusters:
      - name: pytbak
        connect_timeout: 0.25s
        type: LOGICAL_DNS
        dns_lookup_family: V4_ONLY
        lb_policy: ROUND_ROBIN
        load_assignment:
          cluster_name: service_a
          endpoints:
          - lb_endpoints:
            - endpoint:
                address:
                  socket_address:
                    address: localhost
                    port_value: 5000
```

in a Human readable way ...
```
      - name: listener_0
        address:
          socket_address:
            address: 0.0.0.0
            port_value: 5002
```
envoy proxy is listening on port 5002 

```
                - name: local_service
                  domains: ["*"]
                  routes:
                  - match:
                      prefix: "/"
                    route:
                      cluster: pytbak
```
no path or domain configuration , just handle all after ```/```

```
      clusters:
      - name: pytbak
        connect_timeout: 0.25s
        type: LOGICAL_DNS
        dns_lookup_family: V4_ONLY
        lb_policy: ROUND_ROBIN
        load_assignment:
          cluster_name: service_a
          endpoints:
          - lb_endpoints:
            - endpoint:
                address:
                  socket_address:
                    address: localhost
                    port_value: 5000
```
something like *reverse proxy* configuration since the app is listening on port 5000


And now the Rate limiting section 
```
              http_filters:
              - name: envoy.filters.http.local_ratelimit
                typed_config:
                  "@type": type.googleapis.com/envoy.extensions.filters.http.local_ratelimit.v3.LocalRateLimit
                  stat_prefix: http_local_rate_limiter
                  token_bucket:
                    max_tokens: 29
                    tokens_per_fill: 29
                    fill_interval: 1s
                  filter_enabled:
                    runtime_key: local_rate_limit_enabled
                    default_value:
                      numerator: 100
                      denominator: HUNDRED
                  filter_enforced:
                    runtime_key: local_rate_limit_enforced
                    default_value:
                      numerator: 100
                      denominator: HUNDRED
                  response_headers_to_add:
                  - append_action: OVERWRITE_IF_EXISTS_OR_ADD
                    header:
                      key: x-local-rate-limit
                      value: 'true'
                  local_rate_limit_per_downstream_connection: false
```
we have a bucket of 29 token , and every second it re-enable the 29 token in the bucket ... i mean ... 29req/second *almost* :-) 
<br>

### Handson  

Again i changed the rate limit to 2req/second  
```
                  token_bucket:
                    max_tokens: 2
                    tokens_per_fill: 2
                    fill_interval: 1s
```
in kubernetes
```pytbak              pytbak-stable-bd648fd46-nj95m              2/2     Running       0                 13s```   

kubectl describe pod pytbak-stable-bd648fd46-nj95m -n pytbak
```
Name:         pytbak-stable-bd648fd46-nj95m
Namespace:    pytbak
Priority:     0
Node:         instance-20220215-1853/10.0.254.135
Start Time:   Sat, 11 Feb 2023 11:19:45 +0000
Labels:       app=pytbak
              pod-template-hash=bd648fd46
              track=pytbak-stable
Annotations:  cni.projectcalico.org/podIP: 10.1.156.173/32
              cni.projectcalico.org/podIPs: 10.1.156.173/32
              prometheus.io/path: /metrics
              prometheus.io/port: 5000
              prometheus.io/scrape: true
Status:       Running
IP:           10.1.156.173
IPs:
  IP:           10.1.156.173
Controlled By:  ReplicaSet/pytbak-stable-bd648fd46
Containers:
  pytbak:
    Container ID:   containerd://9a54504e73a14746299366e2941b51f7069b6649208329a3f87708b053ef1eaf
    Image:          lgirardi/rest-test-multip:0.6
    Image ID:       docker.io/lgirardi/rest-test-multip@sha256:c94695a04fb3b862bfb576ead460aba3528bd098327f88122d087f48380506dd
    Port:           5000/TCP
    Host Port:      0/TCP
    State:          Running
      Started:      Sat, 11 Feb 2023 11:19:46 +0000
    Ready:          True
    Restart Count:  0
    Limits:
      cpu:     300m
      memory:  250Mi
    Requests:
      cpu:        30m
      memory:     125Mi
    Liveness:     http-get http://:5000/api/ delay=40s timeout=10s period=10s #success=1 #failure=3
    Readiness:    http-get http://:5000/api/ delay=5s timeout=15s period=10s #success=1 #failure=3
    Environment:  <none>
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-xrr7c (ro)
  sidecar:
    Container ID:   containerd://a0d45982aa56704983a89bacd664e5cbb8b38b1df943df92ad879e7c66986c22
    Image:          envoyproxy/envoy:v1.22-latest
    Image ID:       docker.io/envoyproxy/envoy@sha256:3b1e0114dbead3fbd9f561994f3894f5d113a815e023065cabdf0c48d55396ce
    Port:           5002/TCP
    Host Port:      0/TCP
    State:          Running
      Started:      Sat, 11 Feb 2023 11:19:47 +0000
    Ready:          True
    Restart Count:  0
    Limits:
      cpu:     100m
      memory:  150Mi
    Requests:
      cpu:        30m
      memory:     55Mi
    Environment:  <none>
    Mounts:
      /etc/envoy from sidecar-config (ro)
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-xrr7c (ro)
Conditions:
  Type              Status
  Initialized       True
  Ready             True
  ContainersReady   True
  PodScheduled      True
Volumes:
  sidecar-config:
    Type:      ConfigMap (a volume populated by a ConfigMap)
    Name:      pytbakt-configmap
    Optional:  false
  kube-api-access-xrr7c:
    Type:                    Projected (a volume that contains injected data from multiple sources)
    TokenExpirationSeconds:  3607
    ConfigMapName:           kube-root-ca.crt
    ConfigMapOptional:       <nil>
    DownwardAPI:             true
QoS Class:                   Burstable
Node-Selectors:              <none>
Tolerations:                 node.kubernetes.io/not-ready:NoExecute op=Exists for 300s
                             node.kubernetes.io/unreachable:NoExecute op=Exists for 300s
Events:
  Type    Reason     Age    From               Message
  ----    ------     ----   ----               -------
  Normal  Scheduled  3m53s  default-scheduler  Successfully assigned pytbak/pytbak-stable-bd648fd46-nj95m to instance-20220215-1853
  Normal  Pulled     3m53s  kubelet            Container image "lgirardi/rest-test-multip:0.6" already present on machine
  Normal  Created    3m53s  kubelet            Created container pytbak
  Normal  Started    3m53s  kubelet            Started container pytbak
  Normal  Pulled     3m53s  kubelet            Container image "envoyproxy/envoy:v1.22-latest" already present on machine
  Normal  Created    3m53s  kubelet            Created container sidecar
  Normal  Started    3m52s  kubelet            Started container sidecar
  ```


With curl 

$ while true;do curl -I  http://oracolo.k8s.it/api/fib/1 && sleep 0.4;done
```
HTTP/1.1 200 OK
Date: Sat, 11 Feb 2023 11:29:48 GMT
Content-Type: text/html; charset=utf-8
Content-Length: 1
Connection: keep-alive
vary: Accept-Encoding
x-envoy-upstream-service-time: 2

HTTP/1.1 200 OK
Date: Sat, 11 Feb 2023 11:29:49 GMT
Content-Type: text/html; charset=utf-8
Content-Length: 1
Connection: keep-alive
vary: Accept-Encoding
x-envoy-upstream-service-time: 2

HTTP/1.1 429 Too Many Requests
Date: Sat, 11 Feb 2023 11:29:49 GMT
Content-Type: text/plain
Content-Length: 18
Connection: keep-alive
x-local-rate-limit: true

HTTP/1.1 200 OK
Date: Sat, 11 Feb 2023 11:29:50 GMT
Content-Type: text/html; charset=utf-8
Content-Length: 1
Connection: keep-alive
vary: Accept-Encoding
x-envoy-upstream-service-time: 2

HTTP/1.1 200 OK
Date: Sat, 11 Feb 2023 11:29:50 GMT
Content-Type: text/html; charset=utf-8
Content-Length: 1
Connection: keep-alive
vary: Accept-Encoding
x-envoy-upstream-service-time: 2

HTTP/1.1 429 Too Many Requests
Date: Sat, 11 Feb 2023 11:29:50 GMT
Content-Type: text/plain
Content-Length: 18
Connection: keep-alive
x-local-rate-limit: true

HTTP/1.1 200 OK
Date: Sat, 11 Feb 2023 11:29:51 GMT
Content-Type: text/html; charset=utf-8
Content-Length: 1
Connection: keep-alive
vary: Accept-Encoding
x-envoy-upstream-service-time: 2

HTTP/1.1 200 OK
Date: Sat, 11 Feb 2023 11:29:51 GMT
Content-Type: text/html; charset=utf-8
Content-Length: 1
Connection: keep-alive
vary: Accept-Encoding
x-envoy-upstream-service-time: 2

HTTP/1.1 429 Too Many Requests
Date: Sat, 11 Feb 2023 11:29:51 GMT
Content-Type: text/plain
Content-Length: 18
Connection: keep-alive
x-local-rate-limit: true
```
<br>
<br>
Both solution are working, different ways and different impacts on the app.  
In the next episode i will explain why i did this job , 
however, now in case u need a simple way to rate limit you app you can have a sidecar container out of the box.



