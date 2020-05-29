# ClusterDNS

DNS is a built-in Kubernetes service launched automatically using the addon manager cluster add-on.<br>
CoreDNS is a general-purpose authoritative DNS server that can serve as cluster DNS. <br>
kube-dns is the legacy one. <br>
CoreDNS supports the features of kube-dns and more(details to be completed https://kubernetes.io/docs/tasks/administer-cluster/dns-custom-nameservers/).

## CoreDNS is responsible for resolving DNS query inside a cluster
CoreDNS is a service serving DNS with its static clusterIP and port 53:
```
# kubectl get svc kube-dns -n kube-system
NAME             TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)                  AGE
kube-dns         ClusterIP   10.1.0.10     <none>        53/UDP,53/TCP,9153/TCP   10d
```
## How can a Pod use CoreDNS service
1. kubelet is configured (by cluster installers) of flags `--cluster-dns=<dns-service-ip>` and `--cluster-domain=<default-local-domain>` with CoreDNS service ClusterIP and cluster domain.
```
# systemctl status kubelet
● kubelet.service - kubelet: The Kubernetes Node Agent
   Loaded: loaded (/usr/lib/systemd/system/kubelet.service; enabled; vendor preset: disabled)
  Drop-In: /usr/lib/systemd/system/kubelet.service.d
           └─10-kubeadm.conf
   Active: active (running) since Tue 2020-05-19 00:16:53 EDT; 1 weeks 3 days ago
     Docs: https://kubernetes.io/docs/
 Main PID: 826 (kubelet)
    Tasks: 16
   Memory: 83.0M
   CGroup: /system.slice/kubelet.service
           └─826 /usr/bin/kubelet --bootstrap-kubeconfig=/etc/kubernetes/bootstrap-kubelet.conf --kubeconfig=/etc/kubernetes/kubelet.conf --config=/var/lib/kubelet/config.yaml --cgroup-...

May 29 09:20:03 vxlan-vm-111-38 kubelet[826]: E0529 09:20:03.520161     826 summary_sys_containers.go:47] Failed to get system container stats for "/system.slice/docker.service": failed...
```
Check kubelet configuration:
```
# cat /var/lib/kubelet/config.yaml
address: 0.0.0.0
apiVersion: kubelet.config.k8s.io/v1beta1
...
clusterDNS:
- 10.1.0.10
clusterDomain: cluster.local
...
resolvConf: /etc/resolv.conf
...
```
2. When Pod starts, kubelet passes the values of the above flags to each Pod's `/etc/resolve.conf`. As a result, the Pod's nameserver is the CoreDNS. All DNS query from cluster will go to CoreDNS.
```
# kubectl exec nginx-deployment-5fdd9b8fb8-5nhkq cat /etc/resolv.conf
nameserver 10.1.0.10
search default.svc.cluster.local svc.cluster.local cluster.local nimbus-tb.eng.vmware.com. eng.vmware.com.
options ndots:5
```
## What do CoreDNS service related resources look like
```
# kubectl get svc  -n kube-system -l k8s-app=kube-dns
NAME       TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)                  AGE
kube-dns   ClusterIP   10.1.0.10    <none>        53/UDP,53/TCP,9153/TCP   10d
# kubectl get ep  -n kube-system -l k8s-app=kube-dns
NAME       ENDPOINTS                                         AGE
kube-dns   10.2.0.4:53,10.2.0.5:53,10.2.0.4:53 + 3 more...   10d
# kubectl get deployment  -n kube-system -l k8s-app=kube-dns
NAME      READY   UP-TO-DATE   AVAILABLE   AGE
coredns   2/2     2            2           10d
# kubectl get pod  -n kube-system -l k8s-app=kube-dns
NAME                       READY   STATUS    RESTARTS   AGE
coredns-58cc8c89f4-fhm7c   1/1     Running   1          10d
coredns-58cc8c89f4-wcw8c   1/1     Running   1          10d
```
## How does CoreDNS handle DNS queries
### What records a CoreDNS have
#### A/AAAA Record
(https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/)
##### Each service inside a cluster is assigned a DNS name
* Normal Services are assigned a DNS A or AAAA record
my-svc.my-namespace.svc.cluster-domain.example. This resolves to the cluster IP of the Service.
* Headless Services  are also assigned a DNS A or AAAA record
my-svc.my-namespace.svc.cluster-domain.example. This resolves to the set of IPs of the pods selected by the Service. Clients are expected to consume the set or else use standard round-robin selection from the set
##### Pod is assigned a DNS name
* by default, its hostname is the Pod’s `metadata.name` value
* to specify, use `hostname` and `subdomain` fields.
#### SRV records
#### CNAME records
### When querying records CoreDNS have (aka. cluster services, pod)
(To be completed)
### When queryng records CoreDNS doesn't have (aka. external resources)
(To be completed)
```
# kubectl get cm coredns -n kube-system -oyaml
apiVersion: v1
data:
  Corefile: |
    .:53 {
        errors
        health
        ready
        kubernetes cluster.local in-addr.arpa ip6.arpa {
           pods insecure
           fallthrough in-addr.arpa ip6.arpa
           ttl 30
        }
        prometheus :9153
        forward . /etc/resolv.conf
        cache 30
        loop
        reload
        loadbalance
    }
kind: ConfigMap
metadata:
  creationTimestamp: "2020-05-19T03:42:53Z"
  name: coredns
  namespace: kube-system
  resourceVersion: "202"
  selfLink: /api/v1/namespaces/kube-system/configmaps/coredns
  uid: 651e5532-aff6-4f6e-8311-0988acbfd05f
```

