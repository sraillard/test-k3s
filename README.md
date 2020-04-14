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
kubectl create -f https://raw.githubusercontent.com/sraillard/test-k3s/master/install-dashboard-full-admin-nodeport-v2.0.0-rc7.yaml
```

Modifications:
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
