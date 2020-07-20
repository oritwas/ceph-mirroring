# Before starting
Fork this repository. Ensure two clusters have already been deployed. The kubeconfigs must be modified to allow them to be identified with unique contexts. By default, clusters are deployed with the context name of admin. We will modify that to have the example name of *site1* and *site2*. Once that has been done we will export the configuration and validate we have two contexts.

```
sed -i 's/admin/site1/g' site1/auth/kubeconfig
sed -i 's/admin/site2/g' site2/auth/kubeconfig
export KUBECONFIG=/home/user/site1/auth/kubeconfig:/home/user/site2/auth/kubeconfig
oc config get-contexts
CURRENT   NAME    CLUSTER   AUTHINFO   NAMESPACE
*         site1   site1     site1      
          site2   site2     site2      
```

# Application Endpoint
Because two clusters are required for this demonstration a Load Balancer has to be used to route traffic to the cluster running our sample applciation. This Load Balancer could be either be an internal offering such as a F5 or a global load balancing solution such as Cloudflare.

Perform the steps that relate to your Load Balancing solution.

## Global Load Balancer
Define the URL that will be used for the application. For example, a route could be wordpress.example.com. This route would be defined within the Global Load Balacing service. A health check should be established to check the health of the sites. 

NOTE: A TCP check will give you a false positive healthy cluster so it is suggested to use a HTTP check.


```
sed -i 's/wordpress.demo-sysdeseng.com/wordpress.example.com/g' application/wordpress/base/wordpress-route.yaml
```

Commit the new route to your git repository.
```
git commit -am 'changing route'
git push origin master
```


## HAProxy
This should be used only if you have no other load balancer available. An HAProxy instance will run on one of your clusters. It will route traffic as needed between the two clusters depending on where the application is placed.  This solution is not production capable! If the cluster in which HAproxy goes down then the application will not be routable because there is no HAProxy running on the second cluster.

```
cd haproxy
sh build-haproxy-config.sh
```

Commit the new route to your git repository.
```
git commit -am 'changing route'
git push origin master
```

# ceph-mirroring
The goal of this repository is to allow for users to create workflows using available tooling to migrate an application from one cluster to another cluster. This demonstration requires that two clusters have been created and connectivity exists between the two host neworks.

## Ceph mirroring setup
Ensure the following ports are opened between both sites.
* 4500
* 6789
* 3300
* 6800-7300

On both sites create the following objects.
```
oc create -f rook-ceph-mirroring/common.yaml
oc create -f rook-ceph-mirroring/operator-openshift.yaml
oc create -f rook-ceph-mirroring/cluster-1.3.6-pvc.yaml
```

## Establishing mirroring
The following files must be deployed on both clusters to deploy the ceph mirroring objects. A disk is requested using the storageclass. If the *storageclass* is not gp2 modify the file *rook-ceph-mirroring/cluster-1.3.6-pvc.yaml* replacing *gp2* with your *storageclass*. This may differ between clusters as well especially in a Hybrid cloud. One site may have a *storageclass* named *standard* while another may have a storage class named *thin*. Verify the name of the *storageclass* for your cluster before deploying.

```
oc get sc --context west1
NAME            PROVISIONER             RECLAIMPOLICY   VOLUMEBINDINGMODE      ALLOWVOLUMEEXPANSION   AGE
gp2 (default)   kubernetes.io/aws-ebs   Delete          WaitForFirstConsumer   true                   4h35m

oc get sc --context west2
NAME            PROVISIONER             RECLAIMPOLICY   VOLUMEBINDINGMODE      ALLOWVOLUMEEXPANSION   AGE
gp2 (default)   kubernetes.io/aws-ebs   Delete          WaitForFirstConsumer   true                   4h35m
```

(Optional) If required make the following change.

```
vi rook-ceph-mirroring/cluster-1.3.6-pvc.yaml

    volumeClaimTemplate:
      spec:
        storageClassName: gp2
```

### Creating the ceph objects
Deploy the following yamls on both clusters.
```
oc create -f ceph-deployment/common.yaml --context west1
oc create -f ceph-deployment/common.yaml --context west2
```

```
oc create -f ceph-deployment/cluster-1.3.6-pvc.yaml --context west1
oc create -f ceph-deployment/cluster-1.3.6-pvc.yaml --context west2
```

```
oc create -f ceph-deployment/operator-openshift.yaml --context west1
oc create -f ceph-deployment/operator-openshift.yaml --context west2
```

It will take a few minutes for all of the components to deploy and for the disks to be formatted.


### Launch the toolbox
The toolbox pod is required to interact and configure storage. To deploy the toolbox run the following.

```
oc create -f ceph-deployment/post-deploy/toolbox.yaml --context west1
oc create -f ceph-deployment/post-deploy/toolbox.yaml --context west2
```

### Enable the replica pool
Enable the pools from the Ceph Mgr pod by running entering into the Ceph Mgr Pods 

On site1 run the following in the toolbox:
```
rbd mirror pool enable replicapool image
rbd pool init replicapool
```

On site2 run the following in the toolbox:
```
rbd mirror pool enable replicapool image
rbd pool init replicapool
```






# Application deployment and management
Depending on what tooling is available in your clusters the option exists to use the following tools. Follow the directions in the different subdirectories to deploy the required compontents to be used for applciation management.

[Advanced Cluster Management For Kubernetes Steps](./acm-application-configuration)
[ArgoCD](./argo-applications)
[Tekton](./tekton)
[Manual](./application)

