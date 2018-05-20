This page will show how traffic go between pods with envoy & Istio.

## App traffic -> local proxy (Envoy)
Each pod have below containers:
1. app container
2. istio-proxy: Envoy
3. init containers: [proxy_init](https://github.com/istio/istio/blob/master/pilot/docker/Dockerfile.proxy_init)

[proxy_init](https://github.com/istio/istio/blob/master/pilot/docker/Dockerfile.proxy_init) container have [istio_iptables.sh](https://github.com/istio/istio/blob/master/tools/deb/istio-iptables.sh), it will add iptables rules to ensure all application's output traffic could be intercept by envoy. Let's see what happen.

```
# login to istio-proxy container and check its iptables
root@simon-lab-2422010:~/istio-0.7.1# kubectl exec -it productpage-v1-8666ffbd7c-bkd9f -c istio-proxy /bin/bash
istio-proxy@productpage-v1-8666ffbd7c-bkd9f:/$ sudo iptables -L -t nat

Chain PREROUTING (policy ACCEPT)
target     prot opt source               destination         
ISTIO_REDIRECT  all  --  anywhere             anywhere             /* istio/install-istio-prerouting */

Chain INPUT (policy ACCEPT)
target     prot opt source               destination         

Chain OUTPUT (policy ACCEPT)
target     prot opt source               destination         
ISTIO_OUTPUT  tcp  --  anywhere             anywhere             /* istio/install-istio-output */

Chain POSTROUTING (policy ACCEPT)
target     prot opt source               destination         

Chain ISTIO_OUTPUT (1 references)
target     prot opt source               destination         
ISTIO_REDIRECT  all  --  anywhere            !localhost            /* istio/redirect-implicit-loopback */
RETURN     all  --  anywhere             anywhere             owner UID match istio-proxy /* istio/bypass-envoy */
RETURN     all  --  anywhere             localhost            /* istio/bypass-explicit-loopback */
ISTIO_REDIRECT  all  --  anywhere             anywhere             /* istio/redirect-default-outbound */

Chain ISTIO_REDIRECT (3 references)
target     prot opt source               destination         
REDIRECT   tcp  --  anywhere             anywhere             /* istio/redirect-to-envoy-port */ redir ports 15001

```

OUTPUT rules chain are:
```
ISTIO_OUTPUT  tcp  --  anywhere             anywhere             /* istio/install-istio-output */
```
->
```
ISTIO_REDIRECT  all  --  anywhere             anywhere             /* istio/redirect-default-outbound */
```
->
```
REDIRECT   tcp  --  anywhere             anywhere             /* istio/redirect-to-envoy-port */ redir ports 15001
```

Finally, all application's OUTPUT traffic would be forward to port: 15001.

## Enovy -> target Pod

Envoy will call pilot with internals to get {service:endpoints}, just like service call:
```
# Could login to istio-proxy to try below command
curl istio-pilot.istio-system:8080/v1/registration/
```
After envoy get application's request: http://{service name}/xxx, envoy will get service's endpoints' IP (endpoint could be pod or VM).

## How about pod -> pod traffic without Istio & Envoy
We all know k8s have dns to resolve service name to be service IP.
While how traffic could be forward from service IP to real Pod IP?
Still iptable rules:

Login to k8s node to run below command:
```
iptables -L -t nat
```

Then search "reviews", you'll know how traffic from service IP to pods IP.
