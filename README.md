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

__Note :__ Minikube come with something call [Dynamic provisiong and CSI](https://minikube.sigs.k8s.io/docs/reference/persistent_volumes/). It will create for us a PV based on the PVC with declare so with minikube we don't really need to create the PV yaml. As this feature is only on minikube and on real cluster you will need to create the PV, we are creating it.

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

### __Scaling__

In order to use all the power offers by k8s, we could want a scalable Jenkins.  
One of the strongest sides of Jenkins is that it has a scaling feature almost out-of-the-box. Jenkins scaling is based on the master/slaves model, where you have a number of agent instances (called slaves) and one main Jenkins instance (called master), which is responsible mainly for distributing jobs across slaves.

Sounds good right?

There are plenty of options available to implement Jenkins scaling, we will focus on the [Kubernetes plugin](https://github.com/jenkinsci/kubernetes-plugin).
It allows to run dynamic agents in a Kubernetes cluster by creating a Kubernetes Pod for each agent started.  
It give us a lots of benefits:
  - Ability to run many more build
  - Automatically spinning up and removing slaves based on need, which saves costs
  - Distributing the load across different physical machines while keeping required resources available for each specific build

On your Jenkins web ui select and install the kubernetes plugin:
```
Manage Jenkins |> Manage Plugins |> Available |> Kubernetes
```

Now it’s time to configure the plugin:
```
Manage Jenkins |> Manage Nodes and Clouds |> Configure Clouds |> Add a new cloud |> Kubernetes |> Kubernetes Cloud details...
```

![](/assets/k8s-plugin.png)

We need several things for the configuration:
  - Cluster URL
  - Cluster credential
  - Jenkins URL

In order to connect to the cluster we should have some credential (like your kubectl context).We will use a `X.509 Client Certificate` secret type. To add a new secret, go to:
```
Credentials |> global |> Add Credentials
```
Select `X.509 Client Certificate` under the `Kind` dropdown.  
To get the client key, client certificate and Server CA Certificate, we could do:
```sh
cat ~/.minikube/client.key
cat ~/.minikube/client.crt
cat ~/.minikube/ca.crt
```
Then add `minikube` as ID and a description.

![](/assets/jenkins-secret.png)

The following will give us the cluster url, that will indicate in the `Kubernetes URL` field.
```sh
kubectl cluster-info | grep master
```

After fill out our kubernetes url and our minikube credential, the connection test should be successful.

We can see, that the configuration want to default the jenkins url with `http://<NodeIP>:30000` but we want use the node url because the plugin will connect to the master inside the cluster using the port `5000`.
So, we have to modify three things in our cluster:  
- First, we want our deployment to expose port `5000` and apply it. As we persist the `jenkins_home` folder, there is no worry about deploying a new deployment.
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
            - name: jnlp
              containerPort: 5000
          volumeMounts:
            - name: jenkins-home
              mountPath: /var/jenkins_home
      volumes:
        - name: jenkins-home
          persistentVolumeClaim:
            claimName: jenkins-pvc
```
```sh
kubectl apply -f jenkins-deployment.yaml
```
- Secondly, we want a resilient configuration, that means we don't want give directly the NodeIP as it can change. We also need to be able to communicate on port `5000` within our cluster. So, Let's create a service of type `ClusterIP` and expose both port `8080` and `5000`.
```yaml
apiVersion: v1
kind: Service
metadata:
  name: jenkins-cip
  namespace: jenkins
spec:
  type: ClusterIP
  selector:
    app: jenkins
  ports:
  - name: master
    port: 8080
    targetPort: 8080
    protocol: TCP
  - name: jnlp
    port: 5000
    targetPort: 5000
    protocol: TCP
```
```sh
kubectl apply -f jenkins-services.yaml
```

 - Finally, we want another persistent volume for our jenkins workspace (the folder where jenkins magic happens).
```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: jenkins-workspace-pv
  namespace: jenkins
  labels:
    type: local
spec:
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteMany
  hostPath:
    path: "/data/jenkins-data"
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: jenkins-workspace-pvc
  namespace: jenkins
spec:
  storageClassName: standard
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 10Gi
```
```sh
kubectl apply -f jenkins-volumes.yaml
```

We can now fill out the jenkins url with `http://ClusterIP:8080`, to get the ClusterIP part:
```sh
kubectl get svc -n jenkins
```

Then let's add a pod template and fill it out as the following:
![](/assets/pod-template.png)
![](/assets/pod-template-next.png)

We are finnaly ready to try out the plugin.  
Let's create and name our pipeline:
```
New Item |> Pipeline
```

In the pipeline section we will put the following:
```groovy
podTemplate(inheritFrom: 'jnlp-pod', containers: [
    containerTemplate(name: 'nginx', image: 'nginx', ttyEnabled: true, command: 'cat')
  ]) {

    node(POD_LABEL) {

        stage('cat nginx html')
        {
            container('nginx') {
                sh ```
                echo "--- Cat nginx html ---"
                cat "/usr/share/nginx/html/index.html"
                ```
            }
        }
    }
}
```

### __Automation__

The best way to install the jenkins master with our custom plugin is to create a docker image based on the official base image. Having your Jenkins setup as a dockerfile is highly recommended to make your continuous integration infrastructure replicable and have it ready for setup from scratch. To install plugins on the dockerfile:

```Dockerfile
FROM jenkins/jenkins

...
RUN /usr/local/bin/install-plugins.sh kubernetes
...
```