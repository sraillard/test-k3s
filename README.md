k3s notes
=========

Installing k3s
--------------

Tested on CentOS 7.7 with k3s version v1.17.4+k3s1.

Installation without traefik ingress controller:
```
curl -sfL https://get.k3s.io | INSTALL_K3S_EXEC="server --no-deploy traefik" sh
```

Kubeconfig credentials for kubectl:
```
/etc/rancher/k3s/k3s.yaml
```

Created objects:
* Directories:
  - `/etc/rancher/`
  - `/run/k3s/` (temporary)
  - `/var/lib/kubelet/`
  - `/var/lib/rancher/`
* Tools copied in `/usr/local/bin/`:
  - `k3s`
  - `kubectl`, `crictl`, `ctr`
  - `k3s-uninstall.sh`, `k3s-killall.sh`
* Service:
  - `k3s.service` (running and enabled at startup)
* Networks:
  - `cni0`, `flannel.1`

Checks:
* `k3s check-config`
* `systemctl status k3s`
* `journalctl -u k3s`
* `kubectl get node`
* `kubectl get pods -A`
* `kubectl top pods -A`

Installing the dashboard
------------------------

Dashboard installation yaml based on alternative official file:  
https://raw.githubusercontent.com/kubernetes/dashboard/v2.0.0-rc7/aio/deploy/alternative.yaml

More information about the different options is available here:  
https://github.com/kubernetes/dashboard/blob/master/docs/user/installation.md

Installation script:  
```
kubectl apply -f https://raw.githubusercontent.com/sraillard/test-k3s/master/install-dashboard-full-admin-nodeport-v2.0.0-rc7.yaml
```

Modifications between the orginal file `install-dashboard-alternative-v2.0.0-rc7.yaml` and `install-dashboard-full-admin-nodeport-v2.0.0-rc7.yaml`:
* Service is ***directly exposed*** as nodePort on port TCP/30081 for direct web connection!
* ClusterRole `kubernetes-dashboard` is having ***full cluster admin*** rights!
* In the dashboard deployment, change the container arguments:
  - Remove `--enable-insecure-login`
  - Add `--enable-skip-login`
  - Add `--disable-settings-authorizer`

To remove the dashboard, remove its namespace and global resources:
```
kubectl delete namespace kubernetes-dashboard
kubectl delete clusterrolebindings kubernetes-dashboard
kubectl delete clusterroles kubernetes-dashboard
```

Installing the nginx ingress controller
---------------------------------------

Document about installations:  
https://github.com/kubernetes/ingress-nginx/blob/master/docs/deploy/index.md  
Bare metal case:  
https://github.com/kubernetes/ingress-nginx/blob/master/docs/deploy/baremetal.md  
IP source preservation:  
https://kubernetes.io/docs/tutorials/services/source-ip/  

By default, the Nginx ingress controller will created an auto-signed certificate valid for one year starting from the date of installation with `CN=Kubernetes Ingress Controller Fake Certificate`.

There are different options to expose the ingress controller (that is a Nginx daemon):
* Use a ***LoadBalancer*** service: The production ready option but that needs an external network component. k3s is bundled with a simple LoadBalancer component (it creates a simple pod when needed to handle the external traffic). By default, a LoadBalancer is doing NAT, so to preserve the IP source, the option `externalTrafficPolicy: Local` must be set (and pods must be scheduled on nodes receiving the traffic)
* Use the ***HostNetwork***: In that case, service isn't needed, and the container directly bind the listening port on the node. This is the most direct option (a host port and IP can be specified). Nginx ingress needs a service definition or it will input a lot of error messages! So a service definition must be added even if it isn't needed.
* Use a ***NodePort*** service: IP source isn't preserved due to NAT and only high ports can be exposed (ports TCP/80 and TCP/443 can't be exposed)

Regarding k3s and its default LoadBalancer, it seems to support exposing multiple services from different deployments on the same IP address.

Installation option #1 using the ***LoadBalancer***:
* Use the original mandatory file: `kubectl apply -f https://raw.githubusercontent.com/sraillard/test-k3s/master/install-ingress-nginx-mandatory-v0.30.0.yaml`
* Create the LoadBalancer service: `kubectl apply -f https://raw.githubusercontent.com/sraillard/test-k3s/master/install-ingress-nginx-service-loadbalancer-local.yaml`

Installation option #2 using the ***HostNetwork***:
* Use the modified mandatory file: `kubectl apply -f https://raw.githubusercontent.com/sraillard/test-k3s/master/install-ingress-nginx-mandatory-hostnetwork-internaldns-v0.30.0.yaml`
* At the pod level:
  - `hostNetwork: true` was added to use the node network directly
  - `dnsPolicy: ClusterFirstWithHostNet` was added to use the Kubernetes DNS service (to be able tor resolve internal names like services for backends)
* At the container level:
  - `hostPort: 80`  and `hostPort: 443` were added so the port will be bound directly on the host network with address 0.0.0.0 if `hostIP` isn't specified.
* An internal service was added to prevent the pod complaining

Installation option #3 using the ***NodePort***:
* Use the original mandatory file: `kubectl apply -f https://raw.githubusercontent.com/sraillard/test-k3s/master/install-ingress-nginx-mandatory-v0.30.0.yaml`
* Create the NodePort service: `kubectl apply -f https://raw.githubusercontent.com/sraillard/test-k3s/master/install-ingress-nginx-service-nodeport.yaml`

Checks:
* `netstat -lnt4`: list all listenning ports on node in IPv4
* `kubectl get svc -A`: list all services declared

To remove the Nginx ingress controller, remove its namespace and global resources:
```
kubectl delete NameSpace ingress-nginx
kubectl delete ClusterRole nginx-ingress-clusterrole
kubectl delete ClusterRoleBinding nginx-ingress-clusterrole-nisa-binding
```
