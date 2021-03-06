# NSX-T 2.5.x & K8S  - PART 4
[Home Page](https://github.com/dumlutimuralp/k8s-with-nsx-t-2.5.x)

# Table of Contents
[Deploying NSX Components in K8S Cluster](#Deploying-NSX-Components-in-K8S-Cluster)  
[Creating Namespace and Deploying Test Workloads](#Creating-Namespace-and-Deploying-Test-Workloads)   
[Troubleshooting](#Troubleshooting)

# Deploying NSX Components in K8S Cluster
[Back to Table of Contents](https://github.com/dumlutimuralp/k8s-with-nsx-t-2.5.x/tree/master/Part%204#Table-of-Contents)

1. Copy the "nsx-ncp-ubuntu-2.5.0.14628220.tar" file (located in the Kubernetes folder of the .zip file downloaded from my.vmware.com) to a folder on each K8S node. This file is the container image file that is used for containers in the NSX Node Agent Pod and also NSX NCP Bootstracp Pod.  

Hint : Winscp is a handy tool for file copying from Windows to Linux.  

2. On each K8S node, at the prompt, run the <b>"sudo docker load -i nsx-ncp-ubuntu-2.5.0.14628220.tar"</b> command to load the container image to the local Docker container image repository on each K8S node. (This command needs to be run in the same folder which the container image file was copied to on the K8S node) 

3. On each K8S node, tag the image by running <b>"sudo docker tag registry.local/2.5.0.14628220/nsx-ncp-ubuntu:latest nsx-ncp"</b> . This is needed as the container image name used in the manifest file (edited back in Part 3) is pointing out to the container image name of "nsx-ncp". Verify that the image is correctly renamed by running <b>"sudo docker images"</b>.

<pre><code>
root@k8s-master:~# <b>sudo docker images</b>
REPOSITORY                           TAG                 IMAGE ID            CREATED             SIZE
k8s.gcr.io/kube-proxy                v1.14.9             636041c2a488        5 weeks ago         82.1MB
k8s.gcr.io/kube-apiserver            v1.14.9             5811259ed0c9        5 weeks ago         209MB
k8s.gcr.io/kube-controller-manager   v1.14.9             07193a77f264        5 weeks ago         157MB
k8s.gcr.io/kube-scheduler            v1.14.9             0f036524b7a2        5 weeks ago         81.6MB
<b>nsx-ncp</b>                              latest              40aae9a4aeda        3 months ago        744MB
k8s.gcr.io/coredns                   1.3.1               eb516548c180        11 months ago       40.3MB
k8s.gcr.io/etcd                      3.3.10              2c4adeb21b4f        12 months ago       258MB
k8s.gcr.io/pause                     3.1                 da86e6ba6ca1        24 months ago       742kB
root@k8s-master:~#
</code></pre>

4. On K8S master node, from the folder where the "ncp-ubuntu.yaml" was copied to in the node (in the previous post), run "sudo kubectl apply -f ncp-ubuntu.yaml". This command will create a namespace as "nsx-system" in the K8S cluster, then associate a few services accounts and cluster roles in that namespace, then create NCP deployment, NSX NCP Bootstracp daemonset, NSX Node Agent daemonset. 

All of the above can be verified by running "sudo kubectl get all -n nsx-system". As shown below.

<pre><code>
root@k8s-master:~# sudo kubectl get all -n nsx-system
NAME                           READY   STATUS    RESTARTS   AGE
pod/nsx-ncp-848cc8c8ff-k6vfg   1/1     Running   0          13d
pod/nsx-ncp-bootstrap-4mxj5    1/1     Running   0          14d
pod/nsx-ncp-bootstrap-72lvg    1/1     Running   0          14d
pod/nsx-ncp-bootstrap-s5zv4    1/1     Running   0          14d
pod/nsx-node-agent-5xtm4       3/3     Running   0          14d
pod/nsx-node-agent-68ls8       3/3     Running   0          14d
pod/nsx-node-agent-bbpjm       3/3     Running   0          14d

NAME                               DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR   AGE
daemonset.apps/nsx-ncp-bootstrap   3         3         3       3            3           <none>          14d
daemonset.apps/nsx-node-agent      3         3         3       3            3           <none>          14d

NAME                      READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/nsx-ncp   1/1     1            1           14d

NAME                                 DESIRED   CURRENT   READY   AGE
replicaset.apps/nsx-ncp-848cc8c8ff   1         1         1       14d
root@k8s-master:~#
</code></pre>

<b>Note :</b> If any of the Pods, in the above output, show up in "CrashLoopBackOff" status then the logs of the respective container in the respective Pod can be investigated. Please refer to [Troubleshootng](?????) section for more information.

At this stage let' s revisit the topology that was shown in previous posts.

![](2019-12-18_22-26-01.jpg)

Since a K8S cluster comes with multiple namespaces (kube-system, default etc.) and K8S services (kubernetes API, DNS etc.) by default, for these K8S constructs it is expected to have the corresponding NSX-T constructs created automaticaly by NCP. (NCP is the Pod running with the name "nsx-ncp-848cc8c8ff-k6vfg" (in the previous output) in this demonstration. 

Let' s check what has changed on NSX-T side.

A new Tier 1 Gateway has been provisioned with the name "k8s-cluster".

![](2019-12-18_22-34-46.jpg)

Five new segments are provisioned (one per K8S namespace) on the Tier 1 gateway. 

![](2019-12-18_22-35-23.jpg)

IP Pool/address space for each K8S namespace is automatically carved out from the respective IP Block.

![](2019-12-18_23-12-10.jpg)

A new load balancer is provisioned and attached to the Tier 1 Gateway.

![](2019-12-18_22-38-22.jpg)

The VIP for the K8S ingress (Layer 7 LB) is automatically provisioned. The IP adress of the VIP is picked from the "K8S-LB-POOL" automatically.

![](2019-12-18_22-38-40.jpg)

Two firewall rules are automatically provisioned between the sections that were manually configured before (K8S-Begin and K8S-End) . These firewall rules are for K8S node healthcheck traffic destined for CoreDNS Pods.

![](2019-12-18_22-41-42.jpg)

# Creating Namespace and Deploying Test Workloads
[Back to Table of Contents](https://github.com/dumlutimuralp/k8s-with-nsx-t-2.5.x/tree/master/Part%204#Table-of-Contents)

Let' s create a new K8S namespace in this K8S cluster and push a deployment that contains three replicas in it. Following manifest will be used for this purpose. In the simplest form, this manifest creates a namespace, then creates a deployment with a replicaset which contains three replica Pods of "nsxdemo" image. (This yaml is provided [here](https://raw.githubusercontent.com/dumlutimuralp/k8s-with-nsx-t-2.5.x/master/Yaml/demo.yamlhttps://raw.githubusercontent.com/dumlutimuralp/k8s-with-nsx-t-2.5.x/master/Yaml/demo.yaml))

<pre><code>
apiVersion: v1
kind: Namespace
metadata:
 name: demons
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nsxdemo
  namespace: demons
spec:
  selector:
    matchLabels:
      app: nsxdemoapp
  replicas: 3
  template:
    metadata:
      labels:
        app: nsxdemoapp
    spec:
      containers:
      - name: nsx-demo
        image: dumlutimuralp/nsx-demo
        ports:
        - containerPort: 80
</code></pre>

Save the content above in a demo.yaml file on the K8S master node and in the same folder run below command.

<pre><code>
root@k8s-master:~# <b>sudo kubectl create -f demo.yaml</b>
namespace/demons created
deployment.apps/nsxdemo created
root@k8s-master:~#
</code></pre>

Let' s check what has changed in the K8S cluster.

<pre><code>
root@k8s-master:~# sudo kubectl get ns
NAME              STATUS   AGE
default           Active   15d
<b>demons            Active   5m17s</b>
kube-node-lease   Active   15d
kube-public       Active   15d
kube-system       Active   15d
nsx-system        Active   14d
root@k8s-master:~# kubectl get pods -n demons
NAME                       READY   STATUS    RESTARTS   AGE
nsxdemo-65fbb7c8b4-5qwdm   1/1     Running   0          2m24s
nsxdemo-65fbb7c8b4-j5wk5   1/1     Running   0          2m24s
nsxdemo-65fbb7c8b4-rw2b7   1/1     Running   0          2m24s
root@k8s-master:~#
</code></pre>

Let' s see what has changed on the NSX-T side.

A new segment has been created for "demons" K8s namespace. Also all three Pods are connected to the segment.

![](2019-12-19_00-20-56.jpg)

The segment ports has the K8S Pod names built in to the name of the respectve port.

![](2019-12-19_00-24-51.jpg)

All the K8S metadata is also associated with the segment port on NSX-T.

![](2019-12-19_00-26-18.jpg)

A new IP address pool/address space is also assigned to this namespace.

![](2019-12-19_00-27-34.jpg)

The details of the address pool can be reviewed in Advanced UI by nagivating to Advanced Networking & Security tab then -> Inventory -> Groups -> Ip Pools.

# Troubleshooting
[Back to Table of Contents](https://github.com/dumlutimuralp/k8s-with-nsx-t-2.5.x/tree/master/Part%204#Table-of-Contents)

The best way to troubleshoot installation issues or post deployment issues is by looking into the NCP Pod logs or the respective container logs. 

* For instance in the first attempt of this demonstration,all of the of the nsx node agent or nsx ncp bootstrap pods were getting stuck in CrashLoopBackOff state. To troubleshoot the issue one by one the container logs have been investigated. As shown below.

<pre><code>
root@k8s-master:~# sudo kubectl get pods -n nsx-system
NAME                       READY   STATUS    RESTARTS   AGE
nsx-ncp-848cc8c8ff-dlhwq   1/1     Running   0          40m
nsx-ncp-bootstrap-4mxj5    1/1     Running   0          15d
nsx-ncp-bootstrap-72lvg    1/1     Running   0          15d
nsx-ncp-bootstrap-s5zv4    1/1     Running   0          15d
nsx-node-agent-5xtm4       3/3     Running   0          15d
nsx-node-agent-68ls8       3/3     Running   0          15d
nsx-node-agent-bbpjm       3/3     Running   0          15d
root@k8s-master:~# sudo kubectl -n nsx-system logs nsx-node-agent-5xtm4
Error from server (BadRequest): a container name must be specified for pod nsx-node-agent-5xtm4, choose one of: [nsx-node-agent nsx-kube-proxy nsx-ovs]
root@k8s-master:~# sudo kubectl -n nsx-system describe pod/nsx-node-agent-5xtm4
Name:               nsx-node-agent-5xtm4
Namespace:          nsx-system
Priority:           0
PriorityClassName:  <none>
Node:               k8s-worker2/10.190.22.12
Start Time:         Wed, 04 Dec 2019 00:08:10 +0000
Labels:             component=nsx-node-agent
                    controller-revision-hash=7cb5bc85d9
                    pod-template-generation=1
                    tier=nsx-networking
                    version=v1
Annotations:        container.apparmor.security.beta.kubernetes.io/nsx-node-agent: localhost/node-agent-apparmor
Status:             Running
IP:                 10.190.22.12
Controlled By:      DaemonSet/nsx-node-agent
Containers:
 <b>nsx-node-agent</b>:
    Container ID:  docker://0bcebd22cf0f9b820938649e4db5e082730120b52925a9c552b31ff93dd8b59f
    Image:         nsx-ncp
    Image ID:      docker://sha256:40aae9a4aeda247a15d278a844a77cb0cd4361d63e4ce869d2969099ef27f264
----OUTPUT OMITTED----
    Mounts:
      /etc/nsx-ujo/ncp.ini from config-volume (ro,path="ncp.ini")
      /host/proc from proc (ro)
      /var/run/netns from netns (rw)
      /var/run/nsx-ujo from cni-sock (rw)
      /var/run/openvswitch from openvswitch (rw)
      /var/run/secrets/kubernetes.io/serviceaccount from nsx-node-agent-svc-account-token-qj94v (ro)
 <b>nsx-kube-proxy</b>:
    Container ID:  docker://e942905a4efe8f34bf6471a13e4b4bebe656d50c686084f8569d7d6c5b135fbf
    Image:         nsx-ncp
    Image ID:      docker://sha256:40aae9a4aeda247a15d278a844a77cb0cd4361d63e4ce869d2969099ef27f264
----OUTPUT OMITTED----
    Mounts:
      /etc/nsx-ujo/ncp.ini from config-volume (ro,path="ncp.ini")
      /var/run/openvswitch from openvswitch (rw)
      /var/run/secrets/kubernetes.io/serviceaccount from nsx-node-agent-svc-account-token-qj94v (ro)
 <b>nsx-ovs</b>:
    Container ID:  docker://7ac56f3fc4337ee1637537045384f4b58586e9ca56e43645a93f8e93915c1e01
    Image:         nsx-ncp
    Image ID:      docker://sha256:40aae9a4aeda247a15d278a844a77cb0cd4361d63e4ce869d2969099ef27f264
    Port:          <none>
    Host Port:     <none>
    Command:
      start_ovs
----OUTPUT OMITTED----
root@k8s-master:~#
 root@k8s-master:~# <b>kubectl -n nsx-system logs nsx-node-agent-5xtm4 -c nsx-ovs</b>
----OUTPUT OMITTED----
[2019-12-04T22:14:40Z ERROR NSX-OVS]: <b>Please specify ovs_uplink_port under section nsx_node_agent in the ncp.ini config file.</b>
root@k8s-master:~#
----OUTPUT OMITTED----
</code></pre>

According to the error above apparently the ovs uplink port was not configured in the ncp-ubuntu.yaml file manifest.

* Or NCP pod itself may be crashing. Then its logs may need to be reviewed. In most of the cases, if there is anything misspelled or configured in the wrong way in the configmap in the ncp section of the manifest yaml then logs will reveal what that error is. To check the logs of the NCP pod below steps should be followed.

<pre><code>
root@k8s-master:~# sudo kubectl get pods -n nsx-system
NAME                       READY   STATUS    RESTARTS   AGE
nsx-ncp-848cc8c8ff-dlhwq   1/1     Running   0          56m
nsx-ncp-bootstrap-4mxj5    1/1     Running   0          15d
nsx-ncp-bootstrap-72lvg    1/1     Running   0          15d
nsx-ncp-bootstrap-s5zv4    1/1     Running   0          15d
nsx-node-agent-5xtm4       3/3     Running   0          15d
nsx-node-agent-68ls8       3/3     Running   0          15d
nsx-node-agent-bbpjm       3/3     Running   0          15d
root@k8s-master:~# <b>sudo kubectl -n nsx-system logs nsx-ncp-848cc8c8ff-dlhwq --tail=100</b>
----OUTPUT OMITTED----
, u'is_default': False, u'_create_time': 1559050045084, u'transport_type': u'VLAN', u'_protection': u'NOT_PROTECTED', u'host_switch_id': u'44e9c072-d386-46e6-b3b9-d35806ac323b', u'_revision': 0, u'host_switch_mode': u'STANDARD', u'_last_modified_time': 1559050045084, u'_last_modified_user': u'admin', u'id': u'85143f73-6d5c-4603-8651-e89c11d73004', u'resource_type': u'TransportZone'}], u'result_count': 3}
1 2019-12-19T00:57:53.104Z k8s-worker1 NSX 9 - [nsx@6876 comp="nsx-container-ncp" subcomp="ncp" level="INFO"] cli.server.container_cli_server Executed client request "ncp" and sending response on {u'cmd': u'get_k8s_api_server_status', u'id': u'7a7b401f-08fa-451c-9a1a-cbf44e3445e2', u'args': {}} CLI server
----OUTPUT OMITTED----
</code></pre>

# Advanced Networking with K8S
[Back to Table of Contents](https://github.com/dumlutimuralp/k8s-with-nsx-t-2.5.x/tree/master/Part%204#Table-of-Contents)

The [Part 5](https://github.com/dumlutimuralp/k8s-with-nsx-t-2.4.x/tree/master/Part%205) and [Part 6](https://github.com/dumlutimuralp/k8s-with-nsx-t-2.4.x/tree/master/Part%206) of the previous series (K8S with NSX-T 2.4.x) can be reviewed and practiced with NSX-T 2.5.x to learn more about how NSX-T provides native network constructs for K8S workloads.
