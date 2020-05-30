# ClusterDNS

DNS is a built-in Kubernetes service launched automatically using the addon manager cluster add-on.<br>
CoreDNS is a general-purpose authoritative DNS server that can serve as cluster DNS. <br>
kube-dns is the legacy one. <br>
CoreDNS supports the features of kube-dns and more(details to be completed)<br>
(above taken from https://kubernetes.io/docs/tasks/administer-cluster/dns-custom-nameservers/)

## 1. CoreDNS is responsible for resolving DNS query inside a cluster
CoreDNS is a service serving DNS with its static clusterIP and port 53:
```
# kubectl get svc kube-dns -n kube-system
NAME             TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)                  AGE
kube-dns         ClusterIP   10.1.0.10     <none>        53/UDP,53/TCP,9153/TCP   10d
```
## 2. Why a Pod can leverage CoreDNS service
### 2.1. kubelet is configured (by cluster installers) of flags `--cluster-dns=<dns-service-ip>` and `--cluster-domain=<default-local-domain>` with CoreDNS service ClusterIP and cluster domain.
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
kubelet configuration:
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
### 2.2 When Pod starts, kubelet passes the values of the above flags to each Pod's `/etc/resolve.conf`. As a result, the Pod's nameserver is the CoreDNS. All DNS query from cluster will go to CoreDNS.
```
# kubectl exec nginx-deployment-5fdd9b8fb8-5nhkq cat /etc/resolv.conf
nameserver 10.1.0.10
search default.svc.cluster.local svc.cluster.local cluster.local nimbus-tb.eng.vmware.com. eng.vmware.com.
options ndots:5
```
## 3. What CoreDNS service related resources look like
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
## 4. How CoreDNS handles DNS queries
### 4.1 What records a CoreDNS have
#### 4.1.1 A/AAAA Record
(below taken from https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/)
##### Each service inside a cluster is assigned a DNS name
* Normal Services are assigned a DNS A or AAAA record
my-svc.my-namespace.svc.cluster-domain.example. This resolves to the cluster IP of the Service.
* Headless Services  are also assigned a DNS A or AAAA record
my-svc.my-namespace.svc.cluster-domain.example. This resolves to the set of IPs of the pods selected by the Service. Clients are expected to consume the set or else use standard round-robin selection from the set
##### Pod is assigned a DNS name
* by default, its hostname is the Pod’s `metadata.name` value
* to specify, use `hostname` and `subdomain` fields.
#### 4.1.2 SRV records
#### 4.1.3 PTR records
### 4.2 CoreDNS `Corefile` configuration
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
More samples about subdomain and federation: https://kubernetes.io/docs/tasks/administer-cluster/dns-custom-nameservers/
### 4.3 When querying records CoreDNS have (aka. cluster services, pod)
kubernetes: CoreDNS will reply to DNS queries based on IP of the services and pods of Kubernetes
(take from https://kubernetes.io/docs/tasks/administer-cluster/dns-custom-nameservers/) <br>
### 4.4 When queryng records CoreDNS doesn't have (aka. external resources)
forward: Any queries that are not within the cluster domain of Kubernetes will be forwarded to predefined resolvers (/etc/resolv.conf or other upstream DNS server) <br>
(taken from https://kubernetes.io/docs/tasks/administer-cluster/dns-custom-nameservers/)
### 4.5 `/etc/resolv.conf` investigation in CoreDNS Corefile
`/etc/resolv.conf` in coredns pods is same from node DNS config. <br>
By default, kubelet’s `--resolv-conf` flag is set to `/etc/resolv.conf`(please refer to 2.2 kubelet configuration). Add it passes the value to CoreDNS Pods.(a guess from below experiment) <br>
Why CoredNS Pods get same /etc/resolv.conf with node still needs confirm<br>
```
# docker ps |grep coredns
782c4abbf592        bf261d157914                                        "/coredns -conf /etc…"   10 days ago         Up 10 days                                                     k8s_coredns_coredns-58cc8c89f4-wcw8c_kube-system_349ff3a2-8fc8-47c3-a0b2-cd63a92682a5_1
f0c52dd14025        bf261d157914                                        "/coredns -conf /etc…"   10 days ago         Up 10 days                                                     k8s_coredns_coredns-58cc8c89f4-fhm7c_kube-system_a6a7b378-a29a-4650-b4b4-4501ec7ed110_1
5bf863b4f340        registry.aliyuncs.com/google_containers/pause:3.1   "/pause"                 10 days ago         Up 10 days                                                     k8s_PODcoredns-58cc8c89f4-wcw8c_kube-system_349ff3a2-8fc8-47c3-a0b2-cd63a92682a5_3
5067f63e92ad        registry.aliyuncs.com/google_containers/pause:3.1   "/pause"                 10 days ago         Up 10 days                                                     k8s_PODcoredns-58cc8c89f4-fhm7c_kube-system_a6a7b378-a29a-4650-b4b4-4501ec7ed110_3
# cd /var/lib/docker/containers/782c4abbf592a4c70d403a2111795591e3b4fc50627af166a2fc0f5b0c927461/
# grep -r "resolv.conf" .
./config.v2.json:{..."ResolvConfPath":"/var/lib/docker/containers/5bf863b4f340c8e56c6824d39b645bbdb47845e7ce522aa19bc16ec17096b8da/resolv.conf"...}
# cat /var/lib/docker/containers/5bf863b4f340c8e56c6824d39b645bbdb47845e7ce522aa19bc16ec17096b8da/resolv.conf
nameserver 192.168.111.1
search nimbus-tb.eng.vmware.com. eng.vmware.com
(the content in the pause container is same with node's /etc/resolv.conf)
```
## 5. DNS queries policy from a Pod
Kubernetes source code:
```
const (
	// DNSClusterFirstWithHostNet indicates that the pod should use cluster DNS
	// first, if it is available, then fall back on the default
	// (as determined by kubelet) DNS settings.
	DNSClusterFirstWithHostNet DNSPolicy = "ClusterFirstWithHostNet"

	// DNSClusterFirst indicates that the pod should use cluster DNS
	// first unless hostNetwork is true, if it is available, then
	// fall back on the default (as determined by kubelet) DNS settings.
	DNSClusterFirst DNSPolicy = "ClusterFirst"

	// DNSDefault indicates that the pod should use the default (as
	// determined by kubelet) DNS settings.
	DNSDefault DNSPolicy = "Default"

	// DNSNone indicates that the pod should use empty DNS settings. DNS
	// parameters such as nameservers and search paths should be defined via
	// DNSConfig.
	DNSNone DNSPolicy = "None"
)
```
(https://hansedong.github.io/2018/11/20/9/)<br>
* Default
If a Pod’s dnsPolicy is set to “default”, it inherits the name resolution configuration from the node that the Pod runs on (please refer to 2.2 kubelet configuration). The Pod’s DNS resolution should behave the same as the node. You can use the kubelet’s --resolv-conf flag. Set this flag to "" to prevent Pods from inheriting DNS. Set it to a valid file path to specify a file other than /etc/resolv.conf for DNS inheritance. (taken from https://kubernetes.io/docs/tasks/administer-cluster/dns-custom-nameservers/)
* ClusterFirst
A Pod uses cluster DNS aka CoreDNS first, if reslove fails (aka an external hostname query), it uses default upstream DNS settings inherited from node (as determined by kubelet - node's /etc/resolv.conf by default) or other extra stub-domain and upstream DNS servers configured (https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/, https://kubernetes.io/docs/tasks/administer-cluster/dns-custom-nameservers/#effects-on-pods) 

some summaries:<br>
kubelet by default inherited from node DNS settings with `--resolv-conf` flag set to `/etc/resolv.conf`. But we can change to others.<br>
If kubelet a default value, CoreDNS by default uses kubelet value to set its /etc/resolv.conf and Corefile forward. But we can change to others in Corefile.
