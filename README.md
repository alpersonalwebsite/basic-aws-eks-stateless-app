# Deploying a Stateless App

This example consists of the following components:

* A single-instance Redis to store guestbook entries (1 leader to WRITE, multiple followers to READ. Followers sync with the Leader)
* Multiple web frontend instances (2 instances)
* A Load Balancer which is going to expose our app

## Pre reqs

Please, be sure you followed the steps in [Basic AWS EKS](https://github.com/alpersonalwebsite/basic-aws-eks) and you have your cluster up and running.

### TL;DR

**Create cluster and nodegorup**
```shell
eksctl create cluster -f basic-aws-eks/eks/cluster-autoscaling.yaml 
```

This is going to create both stacks:
* eksctl-basic-eks-cluster-cluster
* eksctl-basic-eks-cluster-nodegroup-ng-2

Then, follow the instructions under **Create deployment for AutoScaler**

## Redis

### Create Redis Deployment 

Note: It runs a single replica Redis Pod

```shell
kubectl apply -f eks/redis-leader-deployment.yaml           
```

Output:

```
deployment.apps/redis-leader created
```

### Create Redis Leader Service 

```shell
kubectl apply -f eks/redis-leader-service.yaml 
```

Output:

```
service/redis-leader created
```

We can check the pods that were created:

```shell
kubectl get pods
```

Output:

```
NAME                               READY   STATUS    RESTARTS   AGE
redis-leader-fb76b4755-5f9j6       1/1     Running   0          102s
```

We can also check our service:

```shell
kubectl get service redis-leader
```

Output:

```
NAME           TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
redis-leader   ClusterIP   10.100.67.180   <none>        6379/TCP   2m27s
```

### Create Redis Followers Deployment

We are using 2 replicas to make it highly available.

```shell
kubectl apply -f eks/redis-followers-deployment.yaml 
```

Output:

```
deployment.apps/redis-follower created
```

### Create Redis Follower Service 

```shell
kubectl apply -f eks/redis-follower-service.yaml  
```

Output:

```
service/redis-follower created
```

We can check the pods that were created:

```shell
kubectl get pods
```

Output:

```
NAME                               READY   STATUS    RESTARTS   AGE
redis-follower-dddfbdcc9-cmghr     1/1     Running   0          56s
redis-follower-dddfbdcc9-kk6sc     1/1     Running   0          56s
redis-leader-fb76b4755-5f9j6       1/1     Running   0          7m50s
```

If we want to retrieve more information:

```shell
kubectl get pods -o wide  
```

Output:

```
NAME                               READY   STATUS    RESTARTS   AGE     IP               NODE                                           NOMINATED NODE   READINESS GATES
redis-follower-dddfbdcc9-cmghr     1/1     Running   0          4m47s   192.168.39.202   ip-192-168-36-74.us-west-1.compute.internal    <none>           <none>
redis-follower-dddfbdcc9-kk6sc     1/1     Running   0          4m47s   192.168.21.9     ip-192-168-24-167.us-west-1.compute.internal   <none>           <none>
redis-leader-fb76b4755-5f9j6       1/1     Running   0          11m     192.168.9.221    ip-192-168-24-167.us-west-1.compute.internal   <none>           <none>
```

To describe a node:

```shell
kubectl describe node ip-192-168-36-74.us-west-1.compute.internal
```

Output:

```
Name:               ip-192-168-36-74.us-west-1.compute.internal
Roles:              <none>
Labels:             alpha.eksctl.io/cluster-name=basic-eks-cluster
                    alpha.eksctl.io/instance-id=i-09222615717c99c33
                    alpha.eksctl.io/nodegroup-name=ng-2
                    beta.kubernetes.io/arch=amd64
                    beta.kubernetes.io/instance-type=t2.small
                    beta.kubernetes.io/os=linux
                    failure-domain.beta.kubernetes.io/region=us-west-1
                    failure-domain.beta.kubernetes.io/zone=us-west-1c
                    instance-type=onDemand
                    k8s.io/cloud-provider-aws=4a089614cad06d062dec3a086241b6a9
                    kubernetes.io/arch=amd64
                    kubernetes.io/hostname=ip-192-168-36-74.us-west-1.compute.internal
                    kubernetes.io/os=linux
                    node-lifecycle=on-demand
                    node.kubernetes.io/instance-type=t2.small
                    nodegroup-type=autoscaling
                    topology.kubernetes.io/region=us-west-1
                    topology.kubernetes.io/zone=us-west-1c
Annotations:        node.alpha.kubernetes.io/ttl: 0
                    volumes.kubernetes.io/controller-managed-attach-detach: true
CreationTimestamp:  Mon, 25 Jul 2022 13:56:54 -0700
Taints:             <none>
Unschedulable:      false
Lease:
  HolderIdentity:  ip-192-168-36-74.us-west-1.compute.internal
  AcquireTime:     <unset>
  RenewTime:       Thu, 28 Jul 2022 09:05:57 -0700
Conditions:
  Type             Status  LastHeartbeatTime                 LastTransitionTime                Reason                       Message
  ----             ------  -----------------                 ------------------                ------                       -------
  MemoryPressure   False   Thu, 28 Jul 2022 09:04:33 -0700   Mon, 25 Jul 2022 13:56:54 -0700   KubeletHasSufficientMemory   kubelet has sufficient memory available
  DiskPressure     False   Thu, 28 Jul 2022 09:04:33 -0700   Mon, 25 Jul 2022 13:56:54 -0700   KubeletHasNoDiskPressure     kubelet has no disk pressure
  PIDPressure      False   Thu, 28 Jul 2022 09:04:33 -0700   Mon, 25 Jul 2022 13:56:54 -0700   KubeletHasSufficientPID      kubelet has sufficient PID available
  Ready            True    Thu, 28 Jul 2022 09:04:33 -0700   Mon, 25 Jul 2022 13:57:54 -0700   KubeletReady                 kubelet is posting ready status
Addresses:
  InternalIP:   192.168.36.74
  ExternalIP:   13.57.18.221
  Hostname:     ip-192-168-36-74.us-west-1.compute.internal
  InternalDNS:  ip-192-168-36-74.us-west-1.compute.internal
  ExternalDNS:  ec2-13-57-18-221.us-west-1.compute.amazonaws.com
Capacity:
  attachable-volumes-aws-ebs:  39
  cpu:                         1
  ephemeral-storage:           83873772Ki
  hugepages-2Mi:               0
  memory:                      2028500Ki
  pods:                        11
Allocatable:
  attachable-volumes-aws-ebs:  39
  cpu:                         940m
  ephemeral-storage:           76224326324
  hugepages-2Mi:               0
  memory:                      1541076Ki
  pods:                        11
System Info:
  Machine ID:                 e4a914c9e41f48cf8fbcf96f0c3c2659
  System UUID:                ec27c4ab-d90d-811a-a4e1-f07905500a89
  Boot ID:                    8fc62671-d9a1-46c6-858d-31ed8de40c3c
  Kernel Version:             5.4.196-108.356.amzn2.x86_64
  OS Image:                   Amazon Linux 2
  Operating System:           linux
  Architecture:               amd64
  Container Runtime Version:  docker://20.10.13
  Kubelet Version:            v1.22.9-eks-810597c
  Kube-Proxy Version:         v1.22.9-eks-810597c
ProviderID:                   aws:///us-west-1c/i-09222615717c99c33
Non-terminated Pods:          (6 in total)
  Namespace                   Name                                    CPU Requests  CPU Limits  Memory Requests  Memory Limits  Age
  ---------                   ----                                    ------------  ----------  ---------------  -------------  ---
  default                     redis-follower-dddfbdcc9-cmghr          100m (10%)    0 (0%)      100Mi (6%)       0 (0%)         6m48s
  kube-system                 aws-node-dmk2k                          25m (2%)      0 (0%)      0 (0%)           0 (0%)         2d19h
  kube-system                 kube-proxy-hplfk                        100m (10%)    0 (0%)      0 (0%)           0 (0%)         2d19h
Allocated resources:
  (Total limits may be over 100 percent, i.e., overcommitted.)
  Resource                    Requests     Limits
  --------                    --------     ------
  cpu                         525m (55%)   300m (31%)
  memory                      612Mi (40%)  512Mi (34%)
  ephemeral-storage           0 (0%)       0 (0%)
  hugepages-2Mi               0 (0%)       0 (0%)
  attachable-volumes-aws-ebs  0            0
Events:                       <none>
```

We can also check our service:

```shell
kubectl get service redis-follower
```

Output:

```
NAME             TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)    AGE
redis-follower   ClusterIP   10.100.204.184   <none>        6379/TCP   49s
```

Or get a list of ALL our service:

```shell
kubectl get service  
```

Output:

```
NAME             TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)    AGE
kubernetes       ClusterIP   10.100.0.1       <none>        443/TCP    2d19h
redis-follower   ClusterIP   10.100.204.184   <none>        6379/TCP   103s
redis-leader     ClusterIP   10.100.67.180    <none>        6379/TCP   8m28s
```

## Frontend

### Create Guestbook Deployment

```shell
kubectl apply -f eks/frontend-deployment.yaml 
```

Output:

```
deployment.apps/frontend created
```

We can check the pods that were created:

```shell
kubectl get pods -l app=guestbook -l tier=frontend
```

Output:

```
NAME                        READY   STATUS    RESTARTS   AGE
frontend-85595f5bf9-8456j   1/1     Running   0          45s
frontend-85595f5bf9-bxqtx   1/1     Running   0          45s
frontend-85595f5bf9-tf2q2   1/1     Running   0          45s
```

### Create Guestbook Service

```shell
kubectl apply -f eks/frontend-service.yaml 
```

Output:

```
service/frontend created
```

We can get all the services:

```shell
kubectl get services
```

Output:

```
NAME             TYPE           CLUSTER-IP       EXTERNAL-IP                                                              PORT(S)        AGE
frontend         LoadBalancer   10.100.97.200    a0e2301521a854fa095e671d851db81c-874245712.us-west-1.elb.amazonaws.com   80:30472/TCP   51s
kubernetes       ClusterIP      10.100.0.1       <none>                                                                   443/TCP        3d18h
redis-follower   ClusterIP      10.100.204.184   <none>                                                                   6379/TCP       23h
redis-leader     ClusterIP      10.100.67.180    <none>                                                                   6379/TCP       23h
```

And check all the pods running:

```shell
kubectl get pods -o wide
```

Output:

```
NAME                               READY   STATUS    RESTARTS   AGE     IP               NODE                                           NOMINATED NODE   READINESS GATES
frontend-85595f5bf9-8456j          1/1     Running   0          4m2s    192.168.14.185   ip-192-168-24-167.us-west-1.compute.internal   <none>           <none>
frontend-85595f5bf9-bxqtx          1/1     Running   0          4m2s    192.168.53.244   ip-192-168-36-74.us-west-1.compute.internal    <none>           <none>
frontend-85595f5bf9-tf2q2          1/1     Running   0          4m2s    192.168.9.22     ip-192-168-24-167.us-west-1.compute.internal   <none>           <none>
redis-follower-dddfbdcc9-cmghr     1/1     Running   0          23h     192.168.39.202   ip-192-168-36-74.us-west-1.compute.internal    <none>           <none>
redis-follower-dddfbdcc9-kk6sc     1/1     Running   0          23h     192.168.21.9     ip-192-168-24-167.us-west-1.compute.internal   <none>           <none>
redis-leader-fb76b4755-5f9j6       1/1     Running   0          23h     192.168.9.221    ip-192-168-24-167.us-west-1.compute.internal   <none>           <none>
```

Now, can we start interactiong with our Guestbook application: `a0e2301521a854fa095e671d851db81c-874245712.us-west-1.elb.amazonaws.com`

## Scaling

Currently, we have 3 frontend pods and 3 redis (1 leader and 2 followers) pods running.

If we want to scale, for example, the frontend deployment we can do:

```shell
kubectl scale deployment frontend --replicas=5
```

If we check the pods running:

```
kubectl get pods
```

Output:

```
NAME                               READY   STATUS    RESTARTS   AGE
frontend-85595f5bf9-8456j          1/1     Running   0          13m
frontend-85595f5bf9-bxqtx          1/1     Running   0          13m
frontend-85595f5bf9-dnxp7          1/1     Running   0          30s
frontend-85595f5bf9-kbfbr          1/1     Running   0          30s
frontend-85595f5bf9-tf2q2          1/1     Running   0          13m
redis-follower-dddfbdcc9-cmghr     1/1     Running   0          23h
redis-follower-dddfbdcc9-kk6sc     1/1     Running   0          23h
redis-leader-fb76b4755-5f9j6       1/1     Running   0          23h
```

We can also modify the deployment file. Example: `eks/frontend-deployment.yaml` and then apply `kubectl apply -f eks/frontend-deployment.yaml`

As well using `Kubernetes dashboard`.

## Cleanup

### Frontend

#### Delete Guestbook Deployment

```shell
kubectl delete deployment frontend
```

#### Delete Guestbook Service

```shell
kubectl delete service frontend
```

### Backend

#### Delete Redis Deployment
It will delete both, the leader and the follower

```shell
kubectl delete deployment -l app=redis
```

#### Delete Redis Service
It will delete both, the leader and the follower

```shell
kubectl delete service -l app=redis
```

**The following part is the DELETE section of `basic-aws-eks`**

### Delete nodegroup

This would be the same to `delete in CFN` the stack `eksctl-basic-eks-cluster-nodegroup-ng-2`

```shell
eksctl delete nodegroup --config-file=eks/cluster-autoscaling.yaml --approve 
```

Output:

```
...
...
...
2022-08-09 09:57:33 [✔]  deleted 1 nodegroup(s) from cluster "basic-eks-cluster"
```

### Delete cluster

This would be the same to `delete in CFN` the stack `eksctl-basic-eks-cluster-cluster`

Before doing this, be sure that there are no resources tied to the VPC: example, Security Groups.

```shell
eksctl delete cluster -f eks/cluster-autoscaling.yaml
```

Output:

```shell
2022-08-09 10:02:07 [ℹ]  deleting EKS cluster "basic-eks-cluster"
2022-08-09 10:02:07 [ℹ]  deleted 0 Fargate profile(s)
2022-08-09 10:02:08 [✔]  kubeconfig has been updated
2022-08-09 10:02:08 [ℹ]  cleaning up AWS load balancers created by Kubernetes objects of Kind Service or Ingress
2022-08-09 10:02:09 [ℹ]  1 task: { delete cluster control plane "basic-eks-cluster" [async] }
2022-08-09 10:02:09 [ℹ]  will delete stack "eksctl-basic-eks-cluster-cluster"
2022-08-09 10:02:09 [✔]  all cluster resources were deleted
```

### Delete CFN tacks

```shell
aws cloudformation delete-stack --stack-name 	service-support
```

### Delete user password from parameter store

```shell
aws ssm delete-parameter --name EKSUserPassword --region us-west-1
```

### Delete the EC2 Key Pair

```shell
aws ec2 delete-key-pair --key-name EKSProjectEC2KeyPair --region us-west-1
```