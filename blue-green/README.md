# blue-green deployment demo

Application "wonderful" is running in default Namespace.

You can call the app using `curl wonderful:30080 `.

The application YAML is available at `/wonderful/init.yaml` .

The app has a Deployment with image `httpd:alpine` , but should be switched over to `nginx:alpine` .

The switch should happen instantly. Meaning that from the moment of rollout, all new requests should hit the new image.

Create a new Deployment `wonderful-v2` which uses image `nginx:alpine` with 4 replicas. It's Pods should have labels app: wonderful and version: v2

Once all new Pods are running, change the selector label of Service wonderful to version: v2

Finally scale down Deployment wonderful-v1 to 0 replicas


### Explanation

Why can you call curl wonderful:30080 and it works?


There is a NodePort Service wonderful which listens on port 30080 . It has the Pods of Deployment of app "wonderful" as selector.


We can reach the NodePort Service via the K8s Node IP:


```curl 172.30.1.2:30080```

And because of an entry in /etc/hosts we can call:


```curl wonderful:30080```

####Tip

The idea is to have two Deployments running at the same time, one with the old and one with the new image.

But there is only one Service, and it only points to the Pods of one Deployment.

Once we point the Service to the Pods of the new Deployment, all new requests will hit the new image.


Solution

Create a new Deployment and watch these places:

```
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: wonderful
  name: wonderful-v2
spec:
  replicas: 4
  selector:
    matchLabels:
      app: wonderful
      version: v2
  template:
    metadata:
      labels:
        app: wonderful
        version: v2
    spec:
      containers:
      - image: nginx:alpine
        name: nginx
```

Wait till all Pods are running, then switch the Service selector:

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
    version: v2
  type: NodePort
```
Confirm with curl wonderful:30080 , all requests should hit the new image.


Finally scale down the original Deployment:

```
kubectl scale deploy wonderful-v1 --replicas 0
```
