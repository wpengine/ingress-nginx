# Instructions to resolve backend issue

## Deleting ingress objects and recreating

```shell
# Create deployment with two services: external and internal. The ingress object points to external here.
$ kubectl apply -f test_manifests/example-deployment-multiple-services.yml

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

# Problem fixed
$ curl -H "Host: www.example.com" http://127.0.0.1/
pod1
```

## Updating ingress objects to point to separate service object

```shell
# Create deployment with two services: external and internal. The ingress object points to external here.
$ kubectl apply -f test_manifests/example-deployment-multiple-services.yml

# Curl service - expected behavior
$ curl -H "Host: www.example.com" http://127.0.0.1/
<!doctype html>
<html>
<head>
    <title>Example Domain</title>

# Update ingress object to point to internal service
$ kubectl apply -f test_manifests/ingress-to-internal-service.yml

# Problem fixed
$ curl -H "Host: www.example.com" http://127.0.0.1/
pod1

# Backend successfully reloaded
I0405 14:06:25.528578       8 nginx.go:689] "NGINX configuration change" diff="--- /etc/nginx/nginx.conf\t2022-04-05 14:04:02.271591000 +0000\n+++ /tmp/new-nginx-cfg2551831382\t2022-04-05 14:06:25.525629000 +0000\n@@ -1,5 +1,5 @@\n \n-# Configuration checksum: 8531436188174723858\n+# Configuration checksum: 778490276649772179\n \n # setup custom paths that do not require root access\n pid /tmp/nginx.pid;\n@@ -456,7 +456,7 @@\n \t\t\t\n \t\t\tset $namespace      \"example\";\n \t\t\tset $ingress_name   \"ingress1\";\n-\t\t\tset $service_name   \"external-service\";\n+\t\t\tset $service_name   \"internal-service\";\n \t\t\tset $service_port   \"80\";\n \t\t\tset $location_path  \"/\";\n \t\t\tset $global_rate_limit_exceeding n;\n@@ -500,7 +500,7 @@\n \t\t\tport_in_redirect off;\n \t\t\t\n \t\t\tset $balancer_ewma_score -1;\n-\t\t\tset $proxy_upstream_name \"example-external-service-80\";\n+\t\t\tset $proxy_upstream_name \"example-internal-service-80\";\n \t\t\tset $proxy_host          $proxy_upstream_name;\n \t\t\tset $pass_access_scheme  $scheme;\n \t\t\t\n"
I0405 14:06:25.570908       8 controller.go:176] "Backend successfully reloaded"
I0405 14:06:25.571582       8 event.go:282] Event(v1.ObjectReference{Kind:"Pod", Namespace:"ingress-nginx", Name:"ingress-nginx-controller-7d89b68945-w4jkz", UID:"aae4e247-06b3-4f80-b442-3a7401eb27ec", APIVersion:"v1", ResourceVersion:"9244", FieldPath:""}): type: 'Normal' reason: 'RELOAD' NGINX reload triggered due to a change in configuration
I0405 14:06:25.579179       8 controller.go:201] Dynamic reconfiguration succeeded.
```
