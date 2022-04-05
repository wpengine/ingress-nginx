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
git checkout main



```


## Why fixing removing the cached external backend with the same name is important to us?

Changing to ClusterIP is a dynamic configuration change as oppose to a reload

```log
I0405 13:06:23.398545       7 nginx.go:344] "Event received" type=CREATE object="&Endpoints{ObjectMeta:{service1  example  e88851cf-a68c-472c-9d74-88f8924c74e0 2653 0 2022-04-05 13:06:23 +0000 UTC <nil> <nil> map[app:service1] map[endpoints.kubernetes.io/last-change-trigger-time:2022-04-05T13:01:12Z] [] []  [{kube-controller-manager Update v1 2022-04-05 13:06:23 +0000 UTC FieldsV1 {\"f:metadata\":{\"f:annotations\":{\".\":{},\"f:endpoints.kubernetes.io/last-change-trigger-time\":{}},\"f:labels\":{\".\":{},\"f:app\":{}}},\"f:subsets\":{}} }]},Subsets:[]EndpointSubset{EndpointSubset{Addresses:[]EndpointAddress{EndpointAddress{IP:10.244.0.9,TargetRef:&ObjectReference{Kind:Pod,Namespace:example,Name:pod1-758f994ccb-62t4p,UID:1f8dc83e-fff3-428f-8287-cfa58c060e76,APIVersion:,ResourceVersion:2099,FieldPath:,},Hostname:,NodeName:*ingress-nginx-dev-control-plane,},},NotReadyAddresses:[]EndpointAddress{},Ports:[]EndpointPort{EndpointPort{Name:80-80,Port:80,Protocol:TCP,AppProtocol:nil,},},},},}"
I0405 13:06:23.399454       7 queue.go:87] "queuing" item="&Endpoints{ObjectMeta:{service1  example  e88851cf-a68c-472c-9d74-88f8924c74e0 2653 0 2022-04-05 13:06:23 +0000 UTC <nil> <nil> map[app:service1] map[endpoints.kubernetes.io/last-change-trigger-time:2022-04-05T13:01:12Z] [] []  [{kube-controller-manager Update v1 2022-04-05 13:06:23 +0000 UTC FieldsV1 {\"f:metadata\":{\"f:annotations\":{\".\":{},\"f:endpoints.kubernetes.io/last-change-trigger-time\":{}},\"f:labels\":{\".\":{},\"f:app\":{}}},\"f:subsets\":{}} }]},Subsets:[]EndpointSubset{EndpointSubset{Addresses:[]EndpointAddress{EndpointAddress{IP:10.244.0.9,TargetRef:&ObjectReference{Kind:Pod,Namespace:example,Name:pod1-758f994ccb-62t4p,UID:1f8dc83e-fff3-428f-8287-cfa58c060e76,APIVersion:,ResourceVersion:2099,FieldPath:,},Hostname:,NodeName:*ingress-nginx-dev-control-plane,},},NotReadyAddresses:[]EndpointAddress{},Ports:[]EndpointPort{EndpointPort{Name:80-80,Port:80,Protocol:TCP,AppProtocol:nil,},},},},}"
I0405 13:06:23.413865       7 socket.go:373] "removing metrics" ingresses=[]
I0405 13:06:23.414010       7 nginx_status.go:168] "starting scraping socket" path="/nginx_status"
I0405 13:06:23.433912       7 nginx_status.go:168] "starting scraping socket" path="/nginx_status"
I0405 13:06:23.458647       7 queue.go:128] "syncing" key="example/service1"
I0405 13:06:26.695619       7 controller.go:958] Creating upstream "example-service1-80"
I0405 13:06:26.695666       7 controller.go:1066] Obtaining ports information for Service "example/service1"
I0405 13:06:26.695693       7 endpoints.go:77] Getting Endpoints for Service "example/service1" and port &ServicePort{Name:80-80,Protocol:TCP,Port:80,TargetPort:{0 80 },NodePort:0,AppProtocol:nil,}
I0405 13:06:26.695759       7 endpoints.go:129] Endpoints found for Service "example/service1": [{10.244.0.9 80 &ObjectReference{Kind:Pod,Namespace:example,Name:pod1-758f994ccb-62t4p,UID:1f8dc83e-fff3-428f-8287-cfa58c060e76,APIVersion:,ResourceVersion:2099,FieldPath:,}}]
I0405 13:06:26.695803       7 controller.go:1303] Ingress "example/ingress1" does not contains a TLS section.
I0405 13:06:26.695878       7 controller.go:694] Replacing location "/" for server "www.example.com" with upstream "upstream-default-backend" to use upstream "example-service1-80" (Ingress "example/ingress1")
I0405 13:06:26.695903       7 main.go:175] "Updating ssl expiration metrics"
I0405 13:06:26.695922       7 main.go:180] Updating ssl certificate info metrics
I0405 13:06:26.697111       7 controller.go:201] Dynamic reconfiguration succeeded.
I0405 13:06:26.697314       7 nginx_status.go:168] "starting scraping socket" path="/nginx_status"
I0405 13:06:26.713235       7 socket.go:373] "removing metrics" ingresses=[]
I0405 13:06:26.713533       7 nginx_status.go:168] "starting scraping socket" path="/nginx_status"
I0405 13:06:26.727657       7 nginx_status.go:168] "starting scraping socket" path="/nginx_status"
```