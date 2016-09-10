# kubernetes-faq

This is a random collection of questions and answers I've collected about running and operating kubernetes clusters(primarily AWS). New questions and answers are welcome.

#Contents:

[What happens when a master fails? What happens when a worker fails?](#what-happens-when-a-master-fails-what-happens-when-a-worker-fails)

[How does DNS work in kubernetes?](#how-does-dns-work-in-kubernetes)

[How do I build a High Availability(HA) cluster?](#how-do-i-build-a-high-availabilityha-cluster)

[Should I use Replication Controllers?](#should-i-use-replication-controllers)

[How do I determine the status of a deployment?](#how-do-i-determine-the-status-of-a-deployment)

[How do I rollback a deployment?](#how-do-i-rollback-a-deployment)

[What is a DaemonSet?](#what-is-a-daemonset)

[What is a PetSet?](#what-is-a-petset)

[What is an Ingress Controller?](#what-is-an-ingress-controller)

[How does a kubernetes service work?](#how-does-a-kubernetes-service-work)

[How do I expose a service to a host outside the cluster?](#how-do-i-expose-a-service-to-a-host-outside-the-cluster)

[How do I force a pod to run on a specific node?](#how-do-i-force-a-pod-to-run-on-a-specific-node)

[How do I force replicas of a pod to split across different nodes?](#how-do-i-force-replicas-of-a-pod-to-split-across-different-nodes)

[How can I get the host IP address from inside a pod?](#how-can-i-get-the-host-ip-address-from-inside-a-pod)

[How do I access the kubenernetes API from within a pod?](#how-do-i-access-the-kubenernetes-api-from-within-a-pod)

[Can pods mount NFS volumes?](#can-pods-mount-nfs-volumes)

[Is it possible to route traffic from outside the kubernetes cluster directly to pods?](#is-it-possible-to-route-traffic-from-outside-the-kubernetes-cluster-directly-to-pods)

[How do I put variables into my pods?](#how-do-i-put-variables-into-my-pods)

[Can I use variables or otherwise parameterize my yaml deployment files?](#can-i-use-variables-or-otherwise-parameterize-my-yaml-deployment-files)

[How do CPU and memory requests and limits work?](#how-do-cpu-and-memory-requests-and-limits-work)

[What monitoring and metrics tools do people use for kubernetes?](#what-monitoring-and-metrics-tools-do-people-use-for-kubernetes)

[How should I install kubernetes on AWS?](#how-should-i-install-kubernetes-on-aws)

[How does the default kubernetes AWS networking work?](#how-does-the-default-kubernetes-aws-networking-work)

[How do I add a node to my AWS kubernetes cluster?](#how-do-i-add-a-node-to-my-aws-kubernetes-cluster)

[How do you make a service create a private ELB in AWS instead of the default public one?](#how-do-you-make-a-service-create-a-private-elb-in-aws-instead-of-the-default-public-one)

[How do you restrict an AWS ELB to certain source ips:](#how-do-you-restrict-an-aws-elb-to-certain-source-ips)

[What are some of the AWS limitations?](#what-are-some-of-the-aws-limitations)

[Can you run a multi-AZ kubernetes cluster? What about a multi-region cluster?](#can-you-run-a-multi-az-kubernetes-cluster-what-about-a-multi-region-cluster)

[Is there a way to update route53 DNS with a service members?](#is-there-a-way-to-update-route53-dns-with-a-service-members)

[Can kubernetes auto-create an EBS volume?](#can-kubernetes-auto-create-an-ebs-volume)

[When using an EBS PersistentVolume and PersistentVolumeClaim, how does kubernetes know which AZ to create a pod in?](#when-using-an-ebs-persistentvolume-and-persistentvolumeclaim-how-does-kubernetes-know-which-az-to-create-a-pod-in)

[Does kubernetes support the new Amazon Load Balancer(ALB)?](#does-kubernetes-support-the-new-amazon-load-balanceralb)

[Is there a way to give a pod a separate IAM role that has different permissions than the default instance IAM policy?](#is-there-a-way-to-give-a-pod-a-separate-iam-role-that-has-different-permissions-than-the-default-instance-iam-policy)

[Is kubernetes rack aware or can you detect what region or Availability Zone a host is in?](#is-kubernetes-rack-aware-or-can-you-detect-what-region-or-availability-zone-a-host-is-in)

[Is it possible to install kubernetes into an existing VPC?](#is-it-possible-to-install-kubernetes-into-an-existing-vpc)


# Architecture:

## What happens when a master fails? What happens when a worker fails?

Kubernetes is designed to be resilient to any individual node failure, master or worker. When a master fails the nodes of the cluster will keep operating, but there can be no changes including pod creation or service member changes until the master is available. When a worker fails, the master stops receiving messages from the worker. If the master does not receive status updates from the worker the node will be marked as NotReady. If a node is NotReady for 5 minutes, the master reschedules all pods that were running on the dead node to other available nodes.

## How does DNS work in kubernetes?

There is a DNS server called skydns which runs in a pod in the cluster, in the `kube-system` namespace. That DNS server reads from etcd and can serve up dns entries for kubernetes services to all pods. You can reach any service with the name `<service>.<namespace>.svc.cluster.local`. The resolver automatically searches `<namespace>.svc.cluster.local` dns so that you should be able to call one service to another in the same namespace with just `<service>`.

## How do I build a High Availability(HA) cluster?

The only stateful part of a kubernetes cluster is the etcd. The master server runs the controller manager, scheduler, and the API server and can be run as replicas. The controller manager and scheduler in the master servers use a leader election system, so only one controller manager and scheduler is active for the cluster at any time. So an HA cluster generally consists of an etcd cluster of 3+ nodes and multiple master nodes. http://kubernetes.io/docs/admin/high-availability/#master-elected-components

# Basic usage questions:

## Should I use Replication Controllers? 

Probably not, they are older and have fewer features than the newer deployment objects.

## How do I determine the status of a deployment?

Use `kubectl get deployment <deployment>`. If the `DESIRED`, `CURRENT`, `UP-TO-DATE` are all equal, then the deployment has completed.

## How do I rollback a deployment?

If you apply a change to a deployment with the `--record` flag then kubernetes stores the previous deployment in its history. The `kubectl rollout history deployment <deployment>` command will show prior deployments. The last deployment can be restored with the `kubectl rollout undo deployment <deployment>` command. In progress deployments can also be paused and resumed. http://kubernetes.io/docs/user-guide/kubectl/kubectl_rollout/

## What is a DaemonSet?

A DaemonSet is a set of pods that is run only once on a host. It's used for host-layer features, for instance a network, host monitoring or storage plugin. http://kubernetes.io/docs/admin/daemons/

## What is a PetSet?

In a regular deployment all the instances of a pod are exactly the same, they are indistinguishable and are thus sometimes referred to as "cattle", these are typically stateless applications that can be easily scaled up and down. In a PetSet, each pod is unique and has an identity that needs to be maintained. This is commonly used for more stateful applications like databases. http://kubernetes.io/docs/user-guide/petset/

## What is an Ingress Controller?

An Ingress Controller is a pod that can act as an inbound traffic handler. It is a HTTP reverse proxy that is implemented as a somewhat customizable nginx. Among the features are HTTP path and service based routing and SSL termination. http://kubernetes.io/docs/user-guide/ingress/

## How does a kubernetes service work?

Within the cluster, most kubernetes services are implemented as a virtual IP called a ClusterIP. A ClusterIP has a list of pods which are in the service, the client sends IP traffic directly to a randomly selected pod in the service, so the ClusterIP isn't actually directly routable even from kubernetes nodes. This is all done with iptables routes. The iptables configuration is managed by the kube-proxy on each node. So only nodes running kube-proxy can talk to ClusterIP members. An alternative to the ClusterIP is to use a "Headless" service by specifying ClusterIP=None, this does not use a virtual IP, but instead just creates a DNS A record for a service that includes all the IP addresses of the pods. The live members of any service are stored in an API object called an `endpoint`. You can see the members of a service by doing a `kubectl get endpoints <service>`

## How do I expose a service to a host outside the cluster?

There are two ways, use the NodePort or LoadBalancer service type.

1. NodePort. This makes every node in the cluster listen on the specified NodePort, then any node will forward traffic from that NodePort to a random pod in the service.
1. LoadBalancer. This provisions a NodePort as above, but then does an additional step to provision a load balancer in your cloud(AWS or GKE) automatically. In AWS it also modifies the Auto-Scaling Group of the cluster so all nodes of that ASG are added to the ELB.

## How does a LoadBalancer service work?

A LoadBalancer by default is set up as a TCP Load Balancer with your cloud provider(AWS or GKE). There is no support in bare metal or OpenStack for Load Balancer types. The kubernetes controller manager provisions a load balancer in your cloud and puts all of your kubernetes nodes into the load balancer. Because each node is assumed to be running `kube-proxy` it should be listening on the appropriate NodePort and then it can forward incoming requests to a pod that is available for the service.

Because the LoadBalancer type is by default TCP, not HTTP many higher level features of a LoadBalancer are not available. For instance health checking from the LoadBalancer to the node is done with a TCP check. HTTP X-Forwarded-For information is not available, though it is possible to use proxy protocol in AWS.

http://kubernetes.io/docs/user-guide/services/

## How do I force a pod to run on a specific node?

Kubernetes has node affinity which is described here:
http://kubernetes.io/docs/user-guide/node-selection/

## How do I force replicas of a pod to split across different nodes?

Kubernetes by default does attempt node anti-affinity, but it is not a hard requirement, it is best effort, but will schedule multiple pods on the same node if that is the only way. http://stackoverflow.com/questions/28918056/does-the-kubernetes-scheduler-support-anti-affinity

## How can I get the host IP address from inside a pod?

In kubernetes 1.4 the nodename is available in the downward API in the `spec.nodeName` variable. https://github.com/kubernetes/kubernetes/pull/27880/files .

In earlier versions, you can call the kubernetes API with the appropriate credentials, find the correct API object and extract the hostIP from the output with this convoluted shell command that derives the namespace from the /etc/resolv.conf and gets the pod name from `hostname`:
```
curl -sSk  -H "Authorization: Bearer $(cat /var/run/secrets/kubernetes.io/serviceaccount/token)" \
  https://$KUBERNETES_SERVICE_HOST:$KUBERNETES_PORT_443_TCP_PORT/api/v1/namespaces/$(grep svc.cluster.local /etc/resolv.conf | sed -e 's/search //' -e 's/.svc.cluster.local.*//')/pods/$(hostname) \
  | grep hostIP | grep -oE '[0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3}'
```
The above solution is a "works for me and will probably work for some other people, but could easily fail for a variety of reasons"

In AWS specifically, I find it easier to use the AWS metadata API and just `curl 169.254.169.254/1.0/meta-data/local-ipv4`

## How do I access the kubenernetes API from within a pod?

See the above answer on "getting the host IP address from inside a pod" for an example of using the API inside a pod

## Can pods mount NFS volumes?

Yes, there's an example here of both an NFS client and server running within pods in the cluster: https://github.com/jsafrane/kubernetes-nfs-example

## Is it possible to route traffic from outside the kubernetes cluster directly to pods?

Yes. But one major downside of that is that ClusterIPs are implemented as iptables rules on cluster clients, so you'd lose the ability to see Cluster IPs and service changes. Because the iptables are managed by kube-proxy you could do this by running a kube-proxy, which is similar to just joining the cluster. You could make all your services Headless(ClusterIP = None), this would give your external servers the ability to talk directly to services if they could use the kubernetes dns. Headless services don't use ClusterIPs, but instead just create a DNS A record for all pods in the service. kube-dns is run inside the cluster as a ClusterIP, so there's a chicken and egg problem with DNS you would have to deal with.

## How do I put variables into my pods?

Look at the Downward API: http://kubernetes.io/docs/user-guide/downward-api/
It allows your pods to see labels and annotations and a few other variables using either a mount point or environment variables inside the pod. If those don't contain the information you need, you'll likely need to resort to either using the kubernetes API from within the pod or something entirely separate.

## Can I use variables or otherwise parameterize my yaml deployment files?

There is no built in functionality for this. helm https://github.com/kubernetes/helm is a popular third party choice for this. Some people use scripts or templating tools like jinja.

## How do CPU and memory requests and limits work?

Think of `request` as a node scheduling unit and a `limit` as a hard limit. Kubernetes will attempt to schedule your pods onto nodes based on the sum of the existing requests on the node plus the new pod request. Then setting a limit enforces that cap and makes sure that a pod does not exceed the limit on a node. For simplicity you can just set request and limit to be the same, but you won't be able to pack things tightly onto a node for efficiency. The converse problem is if you use limits which are much larger than requests, there is a danger that pods use resources all the way up to their limit and overrun the node or other pods on the node. CPU may not hard-capped depending on the version of kubernetes/docker, but memory is. Pods exceeding their memory limit will be terminated and rescheduled.

## What monitoring and metrics tools do people use for kubernetes?

Heapster https://github.com/kubernetes/heapster is included and its metrics are how kubernetes measures CPU and memory in order to use horizontal pod autoscaling(hpas). Heapster can be queried directly with its rest API. Prometheus is also more full featured and popular.

## How can containers within a pod communicate with each other?

Containers within a pod share networking space and can reach other on `localhost`. For instance, if you have two containers within a pod, a MySQL container running on port 3306, and a PHP container running on port 80, the PHP container could access the MySQL one through `localhost:3306`.

Read more: https://github.com/kubernetes/kubernetes/blob/release-1.4/docs/design/networking.md#container-to-container


# AWS Questions:

## How should I install kubernetes on AWS?

`kube-up.sh` is being deprecated, it will work for relatively straightforward installs, but won't be actively developed anymore. kops https://github.com/kubernetes/kops is the new recommended deployment tool especially on AWS. It doesn't support other cloud environments yet, but that is planned.

## How does the default kubernetes AWS networking work?

In addition to regular EC2 ip addresses, kubernetes creates its own cluster internal network. In AWS, each instance in a kubernetes cluster hosts a small subnet(by default a /24), and each pod on that host get its own IP addresses within that node's /24 address space. Whenever a cluster node is added or deleted, the kubernetes controller updates the route table so that all nodes in the VPC can route pod IPs directly to that node's subnet.

## How do I add a node to my AWS kubernetes cluster?

If you used `kube-up.sh` or `kops` to provision your cluster, then it created an AutoScaling Group automatically. You can re-scale that with kops, or update the ASG directly, to grow/shrink the cluster. New instances are provisioned for you and should join the cluster automatically(my experience has been it takes 5-7 minutes for nodes to join). 

With `kops` the recommended process is to edit the InstanceGroup(ig) and then update your cluster. `kops` also supports multiple instance groups per cluster so you can have multiple Auto Scaling Groups to run multiple types of instances within your cluster. Spot instances are also supported. https://github.com/kubernetes/kops/blob/master/docs/instance_groups.md

## How do you make a service create a private ELB in AWS instead of the default public one?

Add the following metadata annotation to your LoadBalancer service
```
service.beta.kubernetes.io/aws-load-balancer-internal: 0.0.0.0/0
```
Then delete and re-create the service object and it should be a new private ELB.

## How do you restrict an AWS ELB to certain source ips:

Add the following metadata annotation to your LoadBalancer service with a comma separated list of CIDRs:
```
service.beta.kubernetes.io/load-balancer-source-ranges
```
Each ELB gets its own security group and this annotation will add those CIDR addresses to the allowed source IPs

https://github.com/kubernetes/kubernetes/blob/d95b9238877d5a74895189069121328c16e420f5/pkg/api/service/annotations.go#L27-L34

## How would I get my ELB LoadBalancer to be an HTTP/s LoadBalancer instead of the default TCP?

Add the following metadata annotations to your service
```
service.beta.kubernetes.io/aws-load-balancer-backend-protocol: http
```
or
```
service.beta.kubernetes.io/aws-load-balancer-backend-protocol: https
```
https://github.com/kubernetes/kubernetes/pull/23495

## How would I attach an SSL certificate to my AWS HTTPS ELB?

Add the following metadata annotations to your service
```
service.beta.kubernetes.io/aws-load-balancer-ssl-cert=arn:aws:acm:us-east-1:123456789012:certificate/12345678-1234-1234-1234-123456789012
```

## What are some of the AWS limitations?

There is a route table entry for every instance in the cluster which allows other nodes to route to the pods on that node. AWS has a soft limit of 50 routes per route table and a hard limit of 100 routes per route table. So with the default VPC networking you will run into one of these limits. Using one of the overlay networks(flannel, weave, romana) can get around this limit. And kiops-routing is a planned feature to do this directly from AWS nodes. A planned feature is for all these networking plugins to be drop in additions to an existing cluster.

## Can you run a multi-AZ kubernetes cluster? What about a multi-region cluster?

Yes and "Not out of the box". Provision with kops and specify the AZ's you want and your cluster will be a multi-AZ cluster within a single region. The AutoScalingGroups can add nodes to any region you specify. The planned solution for a multi-region cluster is to build separate clusters in each region and use federation to manage multiple clusters. http://kubernetes.io/docs/admin/federation/. It is also possible to build a multi-region cluster using an overlay network like flannel, calico or weave.

## Is there a way to update route53 DNS with a service members?

Third party project: https://github.com/wearemolecule/route53-kubernetes
Future work in core kops: https://github.com/kubernetes/kops/tree/master/dns-controller

## Can kubernetes auto-create an EBS volume?

Yes. When you declare a PersistentVolumeClaim add the annotation:
```
    volume.alpha.kubernetes.io/storage-class: "foo"
```
And the PersistentVolumeClaim will automatically create a volume for you and delete it when the PersistentVolumeClaim is deleted, "foo" is meaningless, the annotation just needs to be set.

## When using an EBS PersistentVolume and PersistentVolumeClaim, how does kubernetes know which AZ to create a pod in?

It just works. EBS volumes are specific to an Availability Zone, and kubernetes knows which AZ a volume is in. When a new pod needs that volume it the pod is  automatically scheduled in the Availability Zone of the volume.

## Does kubernetes support the new Amazon Load Balancer(ALB)?

Not currently.

## Is there a way to give a pod a separate IAM role that has different permissions than the default instance IAM policy?

This is a third party tool which appears to do this: https://github.com/jtblin/kube2iam

## Is kubernetes rack aware or can you detect what region or Availability Zone a host is in?

A kubernetes node has some useful pieces of information attached as labels here are some examples:
```
    beta.kubernetes.io/instance-type: c3.xlarge
    failure-domain.beta.kubernetes.io/region: us-east-1
    failure-domain.beta.kubernetes.io/zone: us-east-1b
    kubernetes.io/hostname: ip-172-31-0-10.us-east-1.compute.internal
```
You can use these for AZ awareness or attach your own labels and use the Downward API for additional flexibility.

## Is it possible to install kubernetes into an existing VPC?

With kops, this should be possible. But having more than one kubernetes cluster in a VPC is not supported.
