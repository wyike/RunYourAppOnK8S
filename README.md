# RunYourAppOnK8S

### 1. You want to run a APP jenkins

#### Jenkins Docker Image: jenkins/jenkins:lts 
(https://hub.docker.com/r/jenkins/jenkins)<br>

#### Jenkins Serving port: 8080 adn 50000
(https://github.com/istresearch/jenkins/blob/master/README.md)<br>
-p 8080:8080 so the 8080 port in the container receives all requests to port 8080 on the host. Jenkins runs on Tomcat, which uses port 8080 as the default<br>
-p 5000:5000 required to attach slave servers; port 50000 is used to communicate between master and slaves

#### Jenkins Home dir: /var/jenkins_home 
(https://www.jenkins.io/doc/book/installing/#accessing-the-jenkins-home-directory)<br>
Map the home dir to the volumes, then you can access and check the contents of the Jenkins App

### 2. Create a jenkins deployment
A jenkins deployment only supports one replica because jenkins home can only be read/wrote by one jenkins master <br>
A jenkins deployment spec has container iamge: jenkins/jenkins:lts <br>
A jenkins deployment spec has containers ports: 8080 and 50000 <br>
A jenkins deployment spec has container volume mount: /var/jenkins_home <br>

```
kubectl apply -f deployment.yaml
```
A deployment creation brings a replicaset created automatically.

### 3. Create a persistent volume claim for jenkins

#### create a managed-fns-storage (TODO)

#### create a pvc
```
kubectl apply -f pvc.yaml
```

### 4. Create a service for jenkins
```
kubectl apply -f service.yaml
```
A service helps discover the endpoints as backend servers.
![](https://github.com/wyike/RunYourAppOnK8S/blob/master/Ingress.jpeg)

### 5. Create an ingress for jenkins

#### create ingress  controller (TODO)

#### create ingress
```
kubectl apply -f ingress.yaml
```

### 6. Access jenkins
#### 6.1 Access jenkins from jenkins pod
get the pod IP<br>
```
[root@vxlan-vm-111-38 jenkins-k8s]# kubectl get pod -owide
NAME                                      READY   STATUS      RESTARTS   AGE     IP          NODE              NOMINATED NODE   READINESS GATES
jenkins-deployment-687ffcd9c4-7fmpf       1/1     Running     0          7h14m   10.2.1.22   vxlan-vm-111-4    <none>           <none>
```
curl pod IP with the container ports (works from each node since pod is reachable inside a cluster):<br> 
```
[root@vxlan-vm-111-38 jenkins-k8s]# curl http://10.2.1.22:8080
<html><head><meta http-equiv='refresh' content='1;url=/login?from=%2F'/><script>window.location.replace('/login?from=%2F');</script></head><body style='background-color:white; color:white;'>


Authentication required
<!--
You are authenticated as: anonymous
Groups that you are in:
  
Permission you need to have (but didn't): hudson.model.Hudson.Read
 ... which is implied by: hudson.security.Permission.GenericRead
 ... which is implied by: hudson.model.Hudson.Administer
-->

</body></html>                                                                                                               
```
and

```
[root@vxlan-vm-111-38 jenkins-k8s]# curl http://10.2.1.22:50000
Jenkins-Agent-Protocols: JNLP4-connect, Ping
Jenkins-Version: 2.222.3
Jenkins-Session: 966a9424
Client: 10.2.0.0
Server: 10.2.1.22
Remoting-Minimum-Version: 3.14
```
#### 6.2 Access jenkins from jenkins service
get the svc cluster IP
```
[root@vxlan-vm-111-38 jenkins-k8s]# kubectl get svc
NAME                     TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)            AGE
jenkins-master-service   ClusterIP   10.1.128.125   <none>        80/TCP,50000/TCP   7h16m
```
curl the cluster IP with cluster ports (works from each node):
```
[root@vxlan-vm-111-38 jenkins-k8s]# curl  http://10.1.128.125:80
<html><head><meta http-equiv='refresh' content='1;url=/login?from=%2F'/><script>window.location.replace('/login?from=%2F');</script></head><body style='background-color:white; color:white;'>


Authentication required
<!--
You are authenticated as: anonymous
Groups that you are in:
  
Permission you need to have (but didn't): hudson.model.Hudson.Read
 ... which is implied by: hudson.security.Permission.GenericRead
 ... which is implied by: hudson.model.Hudson.Administer
-->

</body></html>                
```
and
```
[root@vxlan-vm-111-38 jenkins-k8s]# curl  http://10.1.128.125:50000
Jenkins-Agent-Protocols: JNLP4-connect, Ping
Jenkins-Version: 2.222.3
Jenkins-Session: 966a9424
Client: 10.2.0.0
Server: 10.2.1.22
Remoting-Minimum-Version: 3.14
```
The reason we can reach cluster IP from each node:<br>
kube-ipvs0 inets are created on each node:
```
kube-ipvs0: <BROADCAST,NOARP> mtu 1500 qdisc noop state DOWN group default 
    link/ether 82:3c:2e:25:d1:19 brd ff:ff:ff:ff:ff:ff
    inet 10.1.0.1/32 brd 10.1.0.1 scope global kube-ipvs0
       valid_lft forever preferred_lft forever
    inet 10.1.0.10/32 brd 10.1.0.10 scope global kube-ipvs0
       valid_lft forever preferred_lft forever
    inet 10.1.77.75/32 brd 10.1.77.75 scope global kube-ipvs0
       valid_lft forever preferred_lft forever
    inet 10.1.24.217/32 brd 10.1.24.217 scope global kube-ipvs0
       valid_lft forever preferred_lft forever
    inet 10.1.128.125/32 brd 10.1.128.125 scope global kube-ipvs0
       valid_lft forever preferred_lft forever
```
and rules are created on each node:
```
[root@vxlan-vm-111-38 jenkins-k8s]# ipvsadm -Ln
IP Virtual Server version 1.2.1 (size=4096)
Prot LocalAddress:Port Scheduler Flags
  -> RemoteAddress:Port           Forward Weight ActiveConn InActConn
...
TCP  10.1.128.125:80 rr
  -> 10.2.1.22:8080               Masq    1      0          0         
TCP  10.1.128.125:50000 rr
  -> 10.2.1.22:50000              Masq    1      0          0         
...
```
#### 6.3 Access jenkins from jenkins ingress
##### 6.3.1 Add DNS entry. Eg in /etc/hosts
```
192.168.111.15 jenkins.example.com
```
(node IP which ingress controller runs on / jenkins ingress rules host)<br>

note: I have used an nimubs isolated env which has a jumper. I need to add entry in /etc/hosts and also configure /etc/squid/squid.conf to make sure jenkins.example.com can be resoved from browser <br>
##### 6.3.2 Curl it
```
curl -H 'Host: jenkins.example.com' http://192.168.111.15
```
192.168.111.15 is the node IP which ingress controller runs on
#### 6.4 Access from browser (final goal)
Input http://jenkins.example.com or jenkins.example.com or jenkins.example.com:80 in browser and go <br>
note: If jenkins.example.com, browser will complement to http://jenkins.example.com:80 by default (add http:// as prefix and suffix :80)
