# Reproducing the issue

## Using branch that does not include our change

```shell
$ git clone https://github.com/wpengine/ingress-nginx.git

$ git checkout recreate-issue-with-cached-external-backend

$ make dev-env

# Increase verbosity to --v=3
$ kubectl -n ingress-nginx edit deployment.apps/ingress-nginx-controller

# Create deployment
$ kubectl apply -f test_manifests/example-deployment.yml

# Display example deployed info
NAME                                      READY   STATUS    RESTARTS   AGE   IP            NODE                              NOMINATED NODE   READINES
S GATES
pod/example-deployment-758f994ccb-mctf4   1/1     Running   0          47m   10.244.0.15   ingress-nginx-dev-control-plane   <none>           <none>

NAME               TYPE           CLUSTER-IP   EXTERNAL-IP       PORT(S)   AGE   SELECTOR
service/service1   ExternalName   <none>       www.example.com   <none>    47m   <none>

NAME                                 READY   UP-TO-DATE   AVAILABLE   AGE   CONTAINERS   IMAGES         SELECTOR
deployment.apps/example-deployment   1/1     1            1           47m   pod1         nginx:latest   app=pod1

NAME                                            DESIRED   CURRENT   READY   AGE   CONTAINERS   IMAGES         SELECTOR
replicaset.apps/example-deployment-758f994ccb   1         1         1       47m   pod1         nginx:latest   app=pod1,pod-template-hash=758f994ccb

NAME                                 CLASS   HOSTS             ADDRESS        PORTS   AGE
ingress.networking.k8s.io/ingress1   nginx   www.example.com   10.96.21.186   80      47m

# Curl service - expected behavior
$ curl -H "Host: www.example.com" http://127.0.0.1/
<!doctype html>
<html>
<head>
    <title>Example Domain</title>
...

# Change service type to ClusterIP
$ kubectl apply -f test_manifests/cluster-ip-service.yml

# Curl service again - NOT expected behavior
$ curl -H "Host: www.example.com" http://127.0.0.1/
<!doctype html>
<html>
<head>
    <title>Example Domain</title>

# Restart controller - to force the cached external backends to be cleared from the Lua table backends_with_external_name
$ kubectl -n ingress-nginx rollout restart deployment ingress-nginx-controller

# Curl service - This is the expected behavior we would hope to see
$ curl -H "Host: www.example.com" http://127.0.0.1/
pod1
```

## Using branch that includes our change

```shell
# Apply changes for balancer.lua to current branch
$ git diff recreate-issue-with-cached-external-backend origin/main rootfs/etc/nginx/lua/balancer.lua > fix.patch && git apply ./fix.patch

# export default variable used by `make dev-env` to recreate image with code changes
export TAG=1.0.0-dev
export REGISTRY=gcr.io/k8s-staging-ingress-nginx
export KIND_CLUSTER_NAME="ingress-nginx-dev"
export DEV_IMAGE=${REGISTRY}/controller:${TAG}

# Recreate controller with updated code and load new image to Kind cluster
$ make build image && kind load docker-image --name="${KIND_CLUSTER_NAME}" "${DEV_IMAGE}"

# Restart controller
kubectl -n ingress-nginx rollout restart deployment ingress-nginx-controller

# Apply changes so that ingress is using external service type
kubectl apply -f test_manifests/example-deployment.yml

# Display example deployed info
$ kubectl -n example get all,ingress -o wide
NAME                                      READY   STATUS    RESTARTS   AGE     IP            NODE                              NOMINATED NODE   READINESS GATES
pod/example-deployment-758f994ccb-mctf4   1/1     Running   0          4m44s   10.244.0.15   ingress-nginx-dev-control-plane   <none>           <none>

NAME               TYPE           CLUSTER-IP   EXTERNAL-IP       PORT(S)   AGE     SELECTOR
service/service1   ExternalName   <none>       www.example.com   <none>    4m44s   <none>

NAME                                 READY   UP-TO-DATE   AVAILABLE   AGE     CONTAINERS   IMAGES         SELECTOR
deployment.apps/example-deployment   1/1     1            1           4m44s   pod1         nginx:latest   app=pod1

NAME                                            DESIRED   CURRENT   READY   AGE     CONTAINERS   IMAGES         SELECTOR
replicaset.apps/example-deployment-758f994ccb   1         1         1       4m44s   pod1         nginx:latest   app=pod1,pod-template-hash=758f994ccb

NAME                                 CLASS   HOSTS             ADDRESS        PORTS   AGE
ingress.networking.k8s.io/ingress1   nginx   www.example.com   10.96.21.186   80      4m44s

# Curl Service - expected behavior - fetches example.com
> curl -H "Host: www.example.com" http://127.0.0.1/
<!doctype html>
<html>
<head>
    <title>Example Domain</title>

# Change Service to ClusterIP

# Display example deployed info
> kubectl -n example get all,ingress -o wide
NAME                                      READY   STATUS    RESTARTS   AGE   IP            NODE                              NOMINATED NODE   READINESS GATES
pod/example-deployment-758f994ccb-mctf4   1/1     Running   0          8m    10.244.0.15   ingress-nginx-dev-control-plane   <none>           <none>

NAME               TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)   AGE   SELECTOR
service/service1   ClusterIP   10.96.62.50   <none>        80/TCP    8m    app=pod1

NAME                                 READY   UP-TO-DATE   AVAILABLE   AGE   CONTAINERS   IMAGES         SELECTOR
deployment.apps/example-deployment   1/1     1            1           8m    pod1         nginx:latest   app=pod1

NAME                                            DESIRED   CURRENT   READY   AGE   CONTAINERS   IMAGES         SELECTOR
replicaset.apps/example-deployment-758f994ccb   1         1         1       8m    pod1         nginx:latest   app=pod1,pod-template-hash=758f994ccb

NAME                                 CLASS   HOSTS             ADDRESS        PORTS   AGE
ingress.networking.k8s.io/ingress1   nginx   www.example.com   10.96.21.186   80      8m

# Curl Service again - now works as expected without having to restart the controller.
> curl -H "Host: www.example.com" http://127.0.0.1/
pod1
```


## Why fixing removing the cached external backend with the same name is important to us?

Changing to ClusterIP is a dynamic reconfiguration. No reload required is adventageous for us.

```log
I0405 15:23:20.504476       7 nginx.go:344] "Event received" type=UPDATE object="&Service{ObjectMeta:{service1  example  5439267f-6eb5-48e8-b471-0e1d60577ce2 18067 0 2022-04-05 15:10:13 +0000 UTC <nil> <nil> map[app:service1] map[kubectl.kubernetes.io/last-applied-configuration:{\"apiVersion\":\"v1\",\"kind\":\"Service\",\"metadata\":{\"annotations\":{},\"creationTimestamp\":null,\"labels\":{\"app\":\"service1\"},\"name\":\"service1\",\"namespace\":\"example\"},\"spec\":{\"ports\":[{\"name\":\"80-80\",\"port\":80,\"protocol\":\"TCP\",\"targetPort\":80}],\"selector\":{\"app\":\"pod1\"},\"type\":\"ClusterIP\"}}\n] [] []  [{kubectl-client-side-apply Update v1 2022-04-05 15:23:20 +0000 UTC FieldsV1 {\"f:metadata\":{\"f:annotations\":{\".\":{},\"f:kubectl.kubernetes.io/last-applied-configuration\":{}},\"f:labels\":{\".\":{},\"f:app\":{}}},\"f:spec\":{\"f:ports\":{\".\":{},\"k:{\\\"port\\\":80,\\\"protocol\\\":\\\"TCP\\\"}\":{\".\":{},\"f:name\":{},\"f:port\":{},\"f:protocol\":{},\"f:targetPort\":{}}},\"f:selector\":{\".\":{},\"f:app\":{}},\"f:sessionAffinity\":{},\"f:type\":{}}} }]},Spec:ServiceSpec{Ports:[]ServicePort{ServicePort{Name:80-80,Protocol:TCP,Port:80,TargetPort:{0 80 },NodePort:0,AppProtocol:nil,},},Selector:map[string]string{app: pod1,},ClusterIP:10.96.147.243,Type:ClusterIP,ExternalIPs:[],SessionAffinity:None,LoadBalancerIP:,LoadBalancerSourceRanges:[],ExternalName:,ExternalTrafficPolicy:,HealthCheckNodePort:0,PublishNotReadyAddresses:false,SessionAffinityConfig:nil,IPFamilyPolicy:*SingleStack,ClusterIPs:[10.96.147.243],IPFamilies:[IPv4],AllocateLoadBalancerNodePorts:nil,LoadBalancerClass:nil,InternalTrafficPolicy:nil,},Status:ServiceStatus{LoadBalancer:LoadBalancerStatus{Ingress:[]LoadBalancerIngress{},},Conditions:[]Condition{},},}"
I0405 15:23:20.505134       7 queue.go:87] "queuing" item="&Service{ObjectMeta:{service1  example  5439267f-6eb5-48e8-b471-0e1d60577ce2 18067 0 2022-04-05 15:10:13 +0000 UTC <nil> <nil> map[app:service1] map[kubectl.kubernetes.io/last-applied-configuration:{\"apiVersion\":\"v1\",\"kind\":\"Service\",\"metadata\":{\"annotations\":{},\"creationTimestamp\":null,\"labels\":{\"app\":\"service1\"},\"name\":\"service1\",\"namespace\":\"example\"},\"spec\":{\"ports\":[{\"name\":\"80-80\",\"port\":80,\"protocol\":\"TCP\",\"targetPort\":80}],\"selector\":{\"app\":\"pod1\"},\"type\":\"ClusterIP\"}}\n] [] []  [{kubectl-client-side-apply Update v1 2022-04-05 15:23:20 +0000 UTC FieldsV1 {\"f:metadata\":{\"f:annotations\":{\".\":{},\"f:kubectl.kubernetes.io/last-applied-configuration\":{}},\"f:labels\":{\".\":{},\"f:app\":{}}},\"f:spec\":{\"f:ports\":{\".\":{},\"k:{\\\"port\\\":80,\\\"protocol\\\":\\\"TCP\\\"}\":{\".\":{},\"f:name\":{},\"f:port\":{},\"f:protocol\":{},\"f:targetPort\":{}}},\"f:selector\":{\".\":{},\"f:app\":{}},\"f:sessionAffinity\":{},\"f:type\":{}}} }]},Spec:ServiceSpec{Ports:[]ServicePort{ServicePort{Name:80-80,Protocol:TCP,Port:80,TargetPort:{0 80 },NodePort:0,AppProtocol:nil,},},Selector:map[string]string{app: pod1,},ClusterIP:10.96.147.243,Type:ClusterIP,ExternalIPs:[],SessionAffinity:None,LoadBalancerIP:,LoadBalancerSourceRanges:[],ExternalName:,ExternalTrafficPolicy:,HealthCheckNodePort:0,PublishNotReadyAddresses:false,SessionAffinityConfig:nil,IPFamilyPolicy:*SingleStack,ClusterIPs:[10.96.147.243],IPFamilies:[IPv4],AllocateLoadBalancerNodePorts:nil,LoadBalancerClass:nil,InternalTrafficPolicy:nil,},Status:ServiceStatus{LoadBalancer:LoadBalancerStatus{Ingress:[]LoadBalancerIngress{},},Conditions:[]Condition{},},}"
I0405 15:23:20.505564       7 queue.go:128] "syncing" key="example/service1"
I0405 15:23:20.506561       7 controller.go:958] Creating upstream "example-service1-80"
I0405 15:23:20.506770       7 controller.go:1066] Obtaining ports information for Service "example/service1"
I0405 15:23:20.507133       7 endpoints.go:77] Getting Endpoints for Service "example/service1" and port &ServicePort{Name:80-80,Protocol:TCP,Port:80,TargetPort:{0 80 },NodePort:0,AppProtocol:nil,}
I0405 15:23:20.508178       7 endpoints.go:129] Endpoints found for Service "example/service1": [{10.244.0.15 80 &ObjectReference{Kind:Pod,Namespace:example,Name:example-deployment-758f994ccb-mctf4,UID:32c397b8-cd34-4bd2-851f-11fa137321f6,APIVersion:,ResourceVersion:16575,FieldPath:,}}]
I0405 15:23:20.508422       7 controller.go:1303] Ingress "example/ingress1" does not contains a TLS section.
I0405 15:23:20.508720       7 controller.go:694] Replacing location "/" for server "www.example.com" with upstream "upstream-default-backend" to use upstream "example-service1-80" (Ingress "example/ingress1")
I0405 15:23:20.509154       7 main.go:175] "Updating ssl expiration metrics"
I0405 15:23:20.509327       7 main.go:180] Updating ssl certificate info metrics
I0405 15:23:20.517307       7 controller.go:201] Dynamic reconfiguration succeeded.
I0405 15:23:20.517864       7 nginx_status.go:168] "starting scraping socket" path="/nginx_status"
I0405 15:23:20.555708       7 socket.go:373] "removing metrics" ingresses=[]
I0405 15:23:20.556723       7 nginx_status.go:168] "starting scraping socket" path="/nginx_status"
I0405 15:23:20.586873       7 nginx_status.go:168] "starting scraping socket" path="/nginx_status"
```

## When deleting ingress object, it causes a reload

```shell
# Create deployment with two Service objects so that ingress can be updated between external and internal
$ kubectl apply -f test_manifests/example-deployment-multiple-services.yml
namespace/example created
deployment.apps/example-deployment created
service/external-service created
service/internal-service created
ingress.networking.k8s.io/ingress1 created

# Display example deployed info
NAME                                      READY   STATUS    RESTARTS   AGE   IP            NODE                              NOMINATED NODE   READINESS GATES
pod/example-deployment-66d77d59b5-tfj9p   1/1     Running   0          97s   10.244.0.19   ingress-nginx-dev-control-plane   <none>           <none>

NAME                       TYPE           CLUSTER-IP     EXTERNAL-IP       PORT(S)   AGE   SELECTOR
service/external-service   ExternalName   <none>         www.example.com   <none>    97s   <none>
service/internal-service   ClusterIP      10.96.24.237   <none>            80/TCP    97s   app=internal-service

NAME                                 READY   UP-TO-DATE   AVAILABLE   AGE   CONTAINERS   IMAGES         SELECTOR
deployment.apps/example-deployment   1/1     1            1           97s   pod1         nginx:latest   app=internal-service

NAME                                            DESIRED   CURRENT   READY   AGE   CONTAINERS   IMAGES         SELECTOR
replicaset.apps/example-deployment-66d77d59b5   1         1         1       97s   pod1         nginx:latest   app=internal-service,pod-template-hash=66d77d59b5

NAME                                 CLASS   HOSTS             ADDRESS        PORTS   AGE
ingress.networking.k8s.io/ingress1   nginx   www.example.com   10.96.21.186   80      97s

# Curl service - expected behavior
$ curl -H "Host: www.example.com" http://127.0.0.1/
<!doctype html>
<html>
<head>
    <title>Example Domain</title>

# Delete ingress when spin to zero
$ kubectl -n example delete ingress ingress1
ingress.networking.k8s.io "ingress1" deleted

# Not found
$ curl -H "Host: www.example.com" http://127.0.0.1/
<html>
<head><title>404 Not Found</title></head>

# Create ingress object that points to internal service
$ kubectl apply -f test_manifests/ingress-to-internal-service.yml

# Problem fixed but nginx reloaded
$ curl -H "Host: www.example.com" http://127.0.0.1/
pod1

I0405 16:01:56.544377       8 nginx.go:689] "NGINX configuration change" diff="--- /etc/nginx/nginx.conf\t2022-04-05 15:59:21.347021000 +0000\n+++ /tmp/new-nginx-cfg3203772636\t2022-04-05 16:01:56.541175000 +0000\n@@ -1,5 +1,5 @@\n \n-# Configuration checksum: 8088934884504482715\n+# Configuration checksum: 17143906046664301839\n \n # setup custom paths that do not require root access\n pid /tmp/nginx.pid;\n@@ -147,7 +147,7 @@\n \t\n \ttypes_hash_max_size             2048;\n \tserver_names_hash_max_size      1024;\n-\tserver_names_hash_bucket_size   64;\n+\tserver_names_hash_bucket_size   32;\n \tmap_hash_bucket_size            64;\n \t\n \tproxy_headers_hash_max_size     512;\n@@ -437,142 +437,6 @@\n \t}\n \t## end server _\n \t\n-\t## start server www.example.com\n-\tserver {\n-\t\tserver_name www.example.com ;\n-\t\t\n-\t\tlisten 80  ;\n-\t\tlisten [::]:80  ;\n-\t\tlisten 443  ssl http2 ;\n-\t\tlisten [::]:443  ssl http2 ;\n-\t\t\n-\t\tset $proxy_upstream_name \"-\";\n-\t\t\n-\t\tssl_certificate_by_lua_block {\n-\t\t\tcertificate.call()\n-\t\t}\n-\t\t\n-\t\tlocation / {\n-\t\t\t\n-\t\t\tset $namespace      \"example\";\n-\t\t\tset $ingress_name   \"ingress1\";\n-\t\t\tset $service_name   \"service1\";\n-\t\t\tset $service_port   \"80\";\n-\t\t\tset $location_path  \"/\";\n-\t\t\tset $global_rate_limit_exceeding n;\n-\t\t\t\n-\t\t\trewrite_by_lua_block {\n-\t\t\t\tlua_ingress.rewrite({\n-\t\t\t\t\tforce_ssl_redirect = false,\n-\t\t\t\t\tssl_redirect = true,\n-\t\t\t\t\tforce_no_ssl_redirect = false,\n-\t\t\t\t\tpreserve_trailing_slash = false,\n-\t\t\t\t\tuse_port_in_redirects = false,\n-\t\t\t\t\tglobal_throttle = { namespace = \"\", limit = 0, window_size = 0, key = { }, ignored_cidrs = { } },\n-\t\t\t\t})\n-\t\t\t\tbalancer.rewrite()\n-\t\t\t\tplugins.run()\n-\t\t\t}\n-\t\t\t\n-\t\t\t# be careful with `access_by_lua_block` and `satisfy any` directives as satisfy any\n-\t\t\t# will always succeed when there's `access_by_lua_block` that does not have any lua code doing `ngx.exit(ngx.DECLINED)`\n-\t\t\t# other authentication method such as basic auth or external auth useless - all requests will be allowed.\n-\t\t\t#access_by_lua_block {\n-\t\t\t#}\n-\t\t\t\n-\t\t\theader_filter_by_lua_block {\n-\t\t\t\tlua_ingress.header()\n-\t\t\t\tplugins.run()\n-\t\t\t}\n-\t\t\t\n-\t\t\tbody_filter_by_lua_block {\n-\t\t\t\tplugins.run()\n-\t\t\t}\n-\t\t\t\n-\t\t\tlog_by_lua_block {\n-\t\t\t\tbalancer.log()\n-\t\t\t\t\n-\t\t\t\tmonitor.call()\n-\t\t\t\t\n-\t\t\t\tplugins.run()\n-\t\t\t}\n-\t\t\t\n-\t\t\tport_in_redirect off;\n-\t\t\t\n-\t\t\tset $balancer_ewma_score -1;\n-\t\t\tset $proxy_upstream_name \"example-service1-80\";\n-\t\t\tset $proxy_host          $proxy_upstream_name;\n-\t\t\tset $pass_access_scheme  $scheme;\n-\t\t\t\n-\t\t\tset $pass_server_port    $server_port;\n-\t\t\t\n-\t\t\tset $best_http_host      $http_host;\n-\t\t\tset $pass_port           $pass_server_port;\n-\t\t\t\n-\t\t\tset $proxy_alternative_upstream_name \"\";\n-\t\t\t\n-\t\t\tclient_max_body_size                    1m;\n-\t\t\t\n-\t\t\tproxy_set_header Host                   $best_http_host;\n-\t\t\t\n-\t\t\t# Pass the extracted client certificate to the backend\n-\t\t\t\n-\t\t\t# Allow websocket connections\n-\t\t\tproxy_set_header                        Upgrade           $http_upgrade;\n-\t\t\t\n-\t\t\tproxy_set_header                        Connection        $connection_upgrade;\n-\t\t\t\n-\t\t\tproxy_set_header X-Request-ID           $req_id;\n-\t\t\tproxy_set_header X-Real-IP              $remote_addr;\n-\t\t\t\n-\t\t\tproxy_set_header X-Forwarded-For        $remote_addr;\n-\t\t\t\n-\t\t\tproxy_set_header X-Forwarded-Host       $best_http_host;\n-\t\t\tproxy_set_header X-Forwarded-Port       $pass_port;\n-\t\t\tproxy_set_header X-Forwarded-Proto      $pass_access_scheme;\n-\t\t\tproxy_set_header X-Forwarded-Scheme     $pass_access_scheme;\n-\t\t\t\n-\t\t\tproxy_set_header X-Scheme               $pass_access_scheme;\n-\t\t\t\n-\t\t\t# Pass the original X-Forwarded-For\n-\t\t\tproxy_set_header X-Original-Forwarded-For $http_x_forwarded_for;\n-\t\t\t\n-\t\t\t# mitigate HTTPoxy Vulnerability\n-\t\t\t# https://www.nginx.com/blog/mitigating-the-httpoxy-vulnerability-with-nginx/\n-\t\t\tproxy_set_header Proxy                  \"\";\n-\t\t\t\n-\t\t\t# Custom headers to proxied server\n-\t\t\t\n-\t\t\tproxy_connect_timeout                   5s;\n-\t\t\tproxy_send_timeout                      60s;\n-\t\t\tproxy_read_timeout                      60s;\n-\t\t\t\n-\t\t\tproxy_buffering                         off;\n-\t\t\tproxy_buffer_size                       4k;\n-\t\t\tproxy_buffers                           4 4k;\n-\t\t\t\n-\t\t\tproxy_max_temp_file_size                1024m;\n-\t\t\t\n-\t\t\tproxy_request_buffering                 on;\n-\t\t\tproxy_http_version                      1.1;\n-\t\t\t\n-\t\t\tproxy_cookie_domain                     off;\n-\t\t\tproxy_cookie_path                       off;\n-\t\t\t\n-\t\t\t# In case of errors try the next upstream server before returning an error\n-\t\t\tproxy_next_upstream                     error timeout;\n-\t\t\tproxy_next_upstream_timeout             0;\n-\t\t\tproxy_next_upstream_tries               3;\n-\t\t\t\n-\t\t\tproxy_pass http://upstream_balancer;\n-\t\t\t\n-\t\t\tproxy_redirect                          off;\n-\t\t\t\n-\t\t}\n-\t\t\n-\t}\n-\t## end server www.example.com\n-\t\n \t# backend for when default-backend-service is not configured or it does not have endpoints\n \tserver {\n \t\tlisten 8181 default_server reuseport backlog=4096;\n"
I0405 16:01:56.583812       8 controller.go:176] "Backend successfully reloaded"
I0405 16:01:56.584645       8 event.go:282] Event(v1.ObjectReference{Kind:"Pod", Namespace:"ingress-nginx", Name:"ingress-nginx-controller-6567f9c9-kc868", UID:"6cf0062d-ba5a-489e-9716-372709762109", APIVersion:"v1", ResourceVersion:"22047", FieldPath:""}): type: 'Normal' reason: 'RELOAD' NGINX reload triggered due to a change in configuration
I0405 16:01:56.595078       8 controller.go:201] Dynamic reconfiguration succeeded.
I0405 16:01:56.595559       8 nginx_status.go:168] "starting scraping socket" path="/nginx_status"
I0405 16:01:56.628993       8 socket.go:373] "removing metrics" ingresses=[example/ingress1]
I0405 16:01:56.630151       8 nginx_status.go:168] "starting scraping socket" path="/nginx_status"
I0405 16:01:56.662331       8 nginx_status.go:168] "starting scraping socket" path="/nginx_status"
```

## Updating the ingress backend without deleting ingress object causes reload

```shell
# Create deployment with two Service objects so that ingress can be updated between external and internal
$ kubectl apply -f test_manifests/example-deployment-multiple-services.yml
namespace/example created
deployment.apps/example-deployment created
service/external-service created
service/internal-service created
ingress.networking.k8s.io/ingress1 created

# Display example deployed info
NAME                                      READY   STATUS    RESTARTS   AGE   IP            NODE                              NOMINATED NODE   READINESS GATES
pod/example-deployment-66d77d59b5-tfj9p   1/1     Running   0          97s   10.244.0.19   ingress-nginx-dev-control-plane   <none>           <none>

NAME                       TYPE           CLUSTER-IP     EXTERNAL-IP       PORT(S)   AGE   SELECTOR
service/external-service   ExternalName   <none>         www.example.com   <none>    97s   <none>
service/internal-service   ClusterIP      10.96.24.237   <none>            80/TCP    97s   app=internal-service

NAME                                 READY   UP-TO-DATE   AVAILABLE   AGE   CONTAINERS   IMAGES         SELECTOR
deployment.apps/example-deployment   1/1     1            1           97s   pod1         nginx:latest   app=internal-service

NAME                                            DESIRED   CURRENT   READY   AGE   CONTAINERS   IMAGES         SELECTOR
replicaset.apps/example-deployment-66d77d59b5   1         1         1       97s   pod1         nginx:latest   app=internal-service,pod-template-hash=66d77d59b5

NAME                                 CLASS   HOSTS             ADDRESS        PORTS   AGE
ingress.networking.k8s.io/ingress1   nginx   www.example.com   10.96.21.186   80      97s

# Update ingress object to point to internal-service
$ kubectl apply -f test_manifests/ingress-to-internal-service.yml
ingress.networking.k8s.io/ingress1 configured

# Changing the ingress object to use different Service object causes reload
I0405 16:12:18.547280       8 nginx.go:689] "NGINX configuration change" diff="--- /etc/nginx/nginx.conf\t2022-04-05 16:10:25.204067000 +0000\n+++ /tmp/new-nginx-cfg2715732543\t2022-04-05 16:12:18.544032000 +0000\n@@ -1,5 +1,5 @@\n \n-# Configuration checksum: 1570461838967710142\n+# Configuration checksum: 14884308888244065627\n \n # setup custom paths that do not require root access\n pid /tmp/nginx.pid;\n@@ -456,7 +456,7 @@\n \t\t\t\n \t\t\tset $namespace      \"example\";\n \t\t\tset $ingress_name   \"ingress1\";\n-\t\t\tset $service_name   \"external-service\";\n+\t\t\tset $service_name   \"internal-service\";\n \t\t\tset $service_port   \"80\";\n \t\t\tset $location_path  \"/\";\n \t\t\tset $global_rate_limit_exceeding n;\n@@ -500,7 +500,7 @@\n \t\t\tport_in_redirect off;\n \t\t\t\n \t\t\tset $balancer_ewma_score -1;\n-\t\t\tset $proxy_upstream_name \"example-external-service-80\";\n+\t\t\tset $proxy_upstream_name \"example-internal-service-80\";\n \t\t\tset $proxy_host          $proxy_upstream_name;\n \t\t\tset $pass_access_scheme  $scheme;\n \t\t\t\n"
I0405 16:12:18.591705       8 controller.go:176] "Backend successfully reloaded"
I0405 16:12:18.592659       8 event.go:282] Event(v1.ObjectReference{Kind:"Pod", Namespace:"ingress-nginx", Name:"ingress-nginx-controller-68c44f9564-5rf2f", UID:"370301a7-3777-4029-9d0f-a2667f4ec121", APIVersion:"v1", ResourceVersion:"23407", FieldPath:""}): type: 'Normal' reason: 'RELOAD' NGINX reload triggered due to a change in configuration
I0405 16:12:18.600450       8 controller.go:201] Dynamic reconfiguration succeeded.
I0405 16:12:18.600744       8 nginx_status.go:168] "starting scraping socket" path="/nginx_status"
I0405 16:12:18.629953       8 socket.go:373] "removing metrics" ingresses=[]
I0405 16:12:18.631009       8 nginx_status.go:168] "starting scraping socket" path="/nginx_status"
I0405 16:12:18.657894       8 nginx_status.go:168] "starting scraping socket" path="/nginx_status"
```
