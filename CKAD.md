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
```
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