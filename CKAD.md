# CKAD notes

### Contexts

kubectl context contains connection information to a K8 cluster. They are normally stored within ~/.kube/config 

```$  kubectl config get-contexts```      this will list all the contexts available 

```$  kubectl config use-context <CONTEXT NAME>```     use the context with CONTEXT NAME 

---
### Resource limits: 

Create a pod template with namespace setting  

```$ kubectl run <POD-NAME>  --image=<IMAGE>   --namespace <NAMESPACE>   --dry-run=client   -o yaml  > pod.yml```


$  edit pod.yml and add the resource requests and limits 

```
... 
containers: 
  - name: app 
    image: IMAGE 
    resources: 
      requests: 
        memory: "64Mi" 
        cpu: "250m" 
      limits: 
        memory: "128Mi" 
        cpu: "500m" 
... 
```
---

### ConfigMaps

Create configmap from key value pairs

``` kubectl create cm <CONFIGMAP-NAME> --from-literal=<KEY>=<VALUE> ```

Create configmap from a file 

``` kubectl create cm <CONFIGMAP-NAME> --from-file=<PROPERTIES-FILE-PATH> ```

Check all the configmaps within the default namespace

``` $ kubectl get cm```

How to mount the config map on a pod/deployment
```
apiVersion: v1
kind: Pod
metadata:
  name: pod1
spec:
  containers:
    - name: pod1
      image: nginx:alpine   
      env:        
        - name: <ENV-KEY>                                      
          valueFrom:
            configMapKeyRef:
              name: <CONFIG-MAP-NAME>          # The ConfigMap name.
              key: <KEY-WITHIN-CONFIG-MAP>     # The key to fetch.        
      volumeMounts:
      - name: <VOLUME-NAME>
        mountPath: "<MOUNT-PATH>"
        readOnly: true
  volumes:
  - name: <VOLUME-NAME>
    configMap:
      name: <CONFIG-MAP-NAME>
        
```
---
### Readiness probes

Create deployment with readiness probes

```
   spec:
      containers:
      - image: httpd:alpine
        name: httpd
        readinessProbe:
          exec:
            command:
              - stat
              - /tmp/ready                  # The command to ensure the pod is ready
          initialDelaySeconds: 10
          periodSeconds: 5
```
---
### Docker basics
Dockerfile
```
FROM bash:latest
CMD ["bash","-c","ping killercoda.com"]
```
Now to build an image with this Dockerfile
```
docker build -t <NAME_OF_IMAGE> .           # If we do not provide a tag with this image, default will be 'latest'
docker build -t pinger .
```
If we have to retag this image
```
docker tag <NAME_OF_IMAGE>:<TAG> <SOME_NEW_NAME>:<NEW_TAG>
docker tag pinger:latest local-registry:5000/pinger:v1
```
To push the image to remote image repository
```30002
docker push <SOME_NEW_NAME>:<NEW_TAG>
docker push local-registry:5000/pinger:v1
```
---
### Rollouts
We can directly edit the deployment and save it to its current annotations
```
kubectl edit deploy/<DEPLOYMENT_NAME> -o yaml --save-config          # save-config will save the modified config in its annotations
```
  #### Rollout GREEN/BLUE
Green blue rollout is where the swtich happens immediately. 
```
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: wonderful
  name: wonderful-v2                        # We renamed it to wonderful-v2
spec:
  replicas: 4
  selector:
    matchLabels:
      app: wonderful
      version: v2                           # changed from v1 to v2
  template:
    metadata:
      labels:
        app: wonderful
        version: v2                         # With the image change we changed v1 to v2
    spec:
      containers:
        - image: nginx:alpine               # Let's say we updated the image from httpd to nginx
          name: httpd
```
We can deploy this above change by using jq , if the manifests are in the single file. This will only use the deployment manifest from the file
```
yq 'select(.kind == "Deployment")' /wonderful/wonderful-v2.yaml | kubectl apply -f -
```
Now we can make change to the service to point towards v2 application
```
apiVersion: v1
kind: Service
metadata:
  labels:
    app: wonderful
  name: wonderful
spec:
  ports:
    - port: 80
      protocol: TCP
      targetPort: 80
      nodePort: 30080
  selector:
    app: wonderful
    version: v2                           # Pointing towards the deployment with version v2
  type: NodePort
```
To deploy the change in the service
```
yq 'select(.kind == "Service")' /wonderful/wonderful-v2.yaml | kubectl apply -f -
```
Since the service is now point towards the newer version of the application we can simply scale down the old deployment
```
kubectl scale deploy wonderful-v1 --replicas=0
```
Switch happens immediately, hence BLUE/GREEN rollout

  #### Rollout Canary

### ReplicaSets
We describe a desired state in a Deployment, and the Deployment Controller changes the actual state of the deployment and underlying replicaSet. A replicaSet creates the underlying pods, ensuring that the desired number of identical pods are stable and running at all times.
```
kubectl create deploy <dep-name> --namespace <namespace> --image=some-image:v1
```
This will create a pod and a replicaset with just 1 replica, which we can scale up and down resulting in updaing the background replicaset
```
kubectl get deploy,pods,rs -n <namespace>
NAME                     READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/dep-name  1/1     1            1           20s

NAME                           READY   STATUS    RESTARTS   AGE
pod/dep-name-58459b8658-6pbmd   1/1     Running   0          20s

NAME                                DESIRED   CURRENT   READY   AGE
replicaset.apps/dep-name-58459b8658   1         1         1       20s
```
Now let's scale the deployment and notice the update on replicaset, which goes to 3
```
kubectl scale deployment scaler --namespace session283884 --replicas 3
kubectl get rs -n <namespace>
NAME                                DESIRED   CURRENT   READY   AGE
replicaset.apps/dep-name-58459b8658   3         3         3       93s
```
## Creating and managing simple k8 resources

### Pod
Command to create a pod within a given namespace
```
kubectl create namespace <namespace>
kubectl run --namespace <namespace> --image=nginx:1.17 <pod-name>
```
Command to delete all the pods within a given namespace
```
kubectl -n <namespace> delete pods --all
```
Command to sort the list of pods with some field value ( sorting in descending order)
```
kubectl get pods -n <namespace> --sort-by=.status.containerStatuses[0].restartCount       ** We are sorting in a descending order by restartCounts.
```

### Deployment
command to create a deployment within a given namespace and with a given number of replicas
```
kubectl create ns <namespace>
kubectl create deploy --namespace <namespace> --image nginx:1.18 --replicas 3 <deploy-name>
```
Scaling up the deployment
```
kubectl scale deploy --replicas=5 -n <namespace> <deploy-name>
```
Update image of the deployment pods
```
kubectl set image --namespace <namespace> deployment/<deploy-name> nginx=nginx:1.19
```
### Service
ClusterIp --> exposes the service but only reachable within the K8 cluster
```
kubectl expose deployment <deply-name> --name=<service-name>  --type=ClusterIP --port=80  --target-port=80  -n <namespace>
```
Confirm the service is targeting the pods with endpoints
```
kubectl get endpoints <service-name> -n <namespace>
NAME          ENDPOINTS                                         AGE
service-name  192.168.0.11:80,192.168.0.57:80,192.168.0.86:80   4m10s
```
NOTE: Endpoints is deprecated in v1.33+ kubernetes versions

