![](/assets/jenkins-x.png )

# __How to setup jenkins on top of a k8s (kubernetes) cluster walkthrough__

To achieve the setup, we will do the following.
  - Setup a k8s sandbox
  - Create a Namespace
  - Create and deploy a Persistent Volume and Persistent Volume Claim yaml
  - Create and deploy the jenkins deployment yaml
  - Create and deploy service yaml
  - Access jenkins
  - To infinity and beyond ...

## __Setup k8s sandbox__

To be able to setup jenkins on k8s cluster, we definitely need a fully runing k8s cluster. As this walkthrough is not about setup a cluster from scratch, we will use a local cluster.  
Please be aware, that is for test purpose only, I would recommend to take a look to the Kubernetes offcial documentation if you want setup a real cluster from scratch.

I will use [Minikube](https://github.com/kubernetes/minikube) tool, which has been developed to run a single node Kubernetes cluster on your local machine. Note that you will have to install a Hypervisor.  
You can also use any other tool you want or even a real cluster, you could have to adapt the Persistent Volume Claim part based on your tool.

The installation step for Windows, Linux or macOS are available [here](https://kubernetes.io/docs/tasks/tools/install-minikube/)

To run minikube after its installation:
```sh
minikube start
```

To confirm that minikube is up and running:
```sh
minikube status
```

To stop minikube:
```sh
minikube stop
```

__Note:__ The kubernetes command line interface, `kubectl`, comes directly with minikube, no needs to install it separately.

## __Create a Namespace__

#### Why should I create a namespace ?

Namespaces allow an isolation in the cluster, we definitely want one on our CI/CD environment. To do so:
```sh
kubectl create ns jenkins
```

Now that we have our namespace, we can start thinking about the deployment.

## __Persistent Volume and Persistent Volume Claim__

#### Why should I need a PV and PVC ?

We want our jenkins to be resilient, we really don't want to loose our setup when the pod crash, restart or when we stop minikube.

As we are on a local single node cluster, we use a hostPath Persistent Volume. According to the documentation,
>*A hostPath PersistentVolume uses a file or directory on the Node to emulate network-attached storage.*

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: jenkins-pv
  namespace: jenkins
  labels:
    type: local
spec:
  storageClassName: manual
  capacity:
    storage: 5Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: "/data/jenkins-data"
```
__Note:__ Minikube is configured to persist files stored under some directories ([list here](https://minikube.sigs.k8s.io/docs/reference/persistent_volumes/)) and `data` is one on them. Change the `path` according your setup.

To request physical storage we need a PersistentVolumeClaim, here is the configuration:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: jenkins-pvc
  namespace: jenkins
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: manual
  resources:
    requests:
      storage: 5Gi
```

Create PV and PVC:
```sh
kubectl apply -f jenkins-volumes.yaml
```
As you can read in the official documentation:  
>*After you create the PersistentVolumeClaim, the Kubernetes control plane looks for a PersistentVolume that satisfies the claim’s requirements. If the control plane finds a suitable PersistentVolume with the same StorageClass, it binds the claim to the volume.*

Let's check that:
```sh
kubectl get pv -n jenkins

NAME         CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                 STORAGECLASS   REASON   AGE
jenkins-pv   5Gi        RWO            Retain           Bound    jenkins/jenkins-pvc   manual                  2m

kubectl get pvc -n jenkins
NAME          STATUS   VOLUME       CAPACITY   ACCESS MODES   STORAGECLASS   AGE
jenkins-pvc   Bound    jenkins-pv   5Gi        RWO            manual         2m
```

Indeed, our PV and PVC are bound together.

## __Create a Jenkins Deployment__