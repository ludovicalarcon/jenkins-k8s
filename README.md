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

Also, we need to install the Kubernetes command line interface to interact witj our cluster.

MacOS
```
curl -LO "https://storage.googleapis.com/kubernetes-release/release/$(curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt)/bin/darwin/amd64/kubectl"

chmod +x kubectl && sudo mv kubectl /usr/local/bin/kubectl
```

Windows (v1.18.0)
```
curl -LO https://storage.googleapis.com/kubernetes-release/release/v1.18.0/bin/windows/amd64/kubectl.exe

Add binary to your PATH
```

Linux
```
curl -LO https://storage.googleapis.com/kubernetes-release/release/`curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt`/bin/linux/amd64/kubectl

chmod +x kubectl && sudo mv kubectl /usr/local/bin/kubectl
```

## __Create Namespace__

#### Why should I need a namespace ?

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
  storageClassName: standard
  accessModes:
    - ReadWriteOnce
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

Perfect, our PV and PVC are bound together.

__Note :__ Minikube come with something call [`Dynamic provisiong and CSI`](https://minikube.sigs.k8s.io/docs/reference/persistent_volumes/). It will create for us a PV based on the PVC with declare so with minikube we don't really need to create the PV yaml. As this feature is only on minikube and on real cluster you will need to create the PV, we are creating it.

## __Create Jenkins Deployment__

Finally we are starting to deploy our jenkins on top of the cluster. The deployment file look like:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: jenkins
  namespace: jenkins
spec:
  replicas: 1
  selector:
    matchLabels:
      app: jenkins
  template:
    metadata:
      labels:
        app: jenkins
    spec:
      containers:
        - name: jenkins
          image: jenkins/jenkins
          ports:
            - name: http-ui
              containerPort: 8080
          volumeMounts:
            - name: jenkins-home
              mountPath: /var/jenkins_home
      volumes:
        - name: jenkins-home
          persistentVolumeClaim:
            claimName: jenkins-pvc
```
As we are in local cluster, we will have only one pod based on the official jenkins' image. As jenkins have its configuration in `/var/jenkins_home` we are mounting our volume on that path. For further information about k8s deployment, you can refer to the [official documentation](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)

Now we apply our deployment:
```sh
kubectl apply -f jenkins-deployment.yaml
```

Let's check if everything is alright with our deployment, you should see the pod in a `running` status.
```sh
kubectl get pods -n jenkins
```

## __Create Service__

#### I alreday have my deployment, why should I need to a service ?

Currently, our jenkins is not accessible to the outside world only within the cluster. Not really useful righ ?
So, for accessing the jenkins container from outside world, we should create a service and map it to the deployment.  
More details about service [here](https://kubernetes.io/docs/concepts/services-networking/service/#publishing-services-service-types)

Our service looks like:
```yaml
apiVersion: v1
kind: Service
metadata:
  name: jenkins
  namespace: jenkins
spec:
  type: NodePort
  selector:
    app: jenkins
  ports:
    - name: http-ui
      port: 8080
      targetPort: 8080
      nodePort: 30000
```

In this, we are using the type `NodePort` which will expose jenkins on all kubernetes node IP's on port `30000`.

Let's create it:
```sh
kubectl apply -f jenkins-services.yaml

// Get NodeIp
kubectl get node -o wide
```

We can acces it by requesting `http://<NodeIP>:30000`

## __Access Jenkins__

Jenkins will ask for initial Admin password, which you can get from the pod logs.

![](/assets/jenkins-lock.png)

```sh
kubectl logs <PodName> -n jenkins

...
*************************************************************
*************************************************************
*************************************************************

Jenkins initial setup is required. An admin user has been created and a password generated.
Please use the following password to proceed to installation:

47a41b5b68b42e39247479d8b9a0c69

This may also be found at: /var/jenkins_home/secrets/initialAdminPassword

*************************************************************
*************************************************************
*************************************************************
...
```

Then install the recommanded plugins and create your admin user.  
You can now start using jenkins and create your pipeline.

![](/assets/jenkins.png)

## __To infinity and beyond ...__