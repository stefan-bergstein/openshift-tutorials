# K8S Basics on OpenShift

```
oc new-project k8s-basics
```

## Quick notes

Start non-root nginx via cli

```
kubectl run nginx --image=bitnami/nginx
```

Check the web server from inside of the pod:
```
kubectl exec nginx -- curl -s http://localhost:8080
```


Start ubi shell container via cli
```
kubectl run shell --image=registry.access.redhat.com/ubi8/ubi --restart=Never --command -- sleep 100000
```

Show labels
```
kubectl get pods --show-labels
```


Set label
```
kubectl label pod nginx  app=basics --overwrite=true
```

Service:
```yaml
kind: Service
apiVersion: v1
metadata:
  name: nginx-service
spec:
  ports:
    - name: web
      protocol: TCP
      port: 8080
      targetPort: 8080
  selector:
    app: basics
```

Check the web server from inside of the nginx pod:
```
kubectl exec shell -- curl -s http://nginx-service:8080
```


Create Ingress:
```yaml
kind: Ingress
apiVersion: networking.k8s.io/v1beta1
metadata:
  name: nginx-ingress
  namespace: k8s-basics
spec:
  rules:
    - host: nginx-ingress.apps-crc.testing
      http:
        paths:
          - path: /
            pathType: ImplementationSpecific
            backend:
              serviceName: nginx-service
              servicePort: 8080
```

Or create a OpenShift route:
```
oc expose svc nginx-service
```


Nginx pod yaml:
```yaml
kind: Pod
apiVersion: v1
metadata:
  name: nginx2
  namespace: k8s-basics
  labels:
    app: basics
spec:
  containers:
  - name: nginx
    image: bitnami/nginx
    ports:
    - containerPort: 8080
```



Nginx pod yaml incl shell ubi container:
```yaml
kind: Pod
apiVersion: v1
metadata:
  name: nginx2
  namespace: k8s-basics
  labels:
    app: basics
spec:
  containers:
  - name: nginx
    image: bitnami/nginx
    ports:
    - containerPort: 8080
  - name: shell
    image: registry.access.redhat.com/ubi8/ubi
    command:
    - "bin/bash"
    - "-c"
    - "sleep 10000"
```


Simple deployment:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  namespace: k8s-basics
spec:
  selector:
    matchLabels:
      app: basics
  replicas: 1
  template:
    metadata:
      labels:
        app: basics
    spec:
      containers:
        - name: nginx
          image: bitnami/nginx
          ports:
            - containerPort: 8080
```


Deployment with init container:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  namespace: k8s-basics
spec:
  selector:
    matchLabels:
      app: basics
  replicas: 1
  template:
    metadata:
      labels:
        app: basics
    spec:
      containers:
        - name: nginx
          image: bitnami/nginx
          ports:
          - containerPort: 8080
          volumeMounts:
          - mountPath: /app
            name: www-data
            readOnly: true
      initContainers:
      - name: ubi-curl
        image: registry.access.redhat.com/ubi8/ubi
        command: ['bin/bash', '-c', "curl https://raw.githubusercontent.com/stefan-bergstein/openshift-tutorials/main/k8s-basics/index.html -o /www/index.html"]
        volumeMounts:
        - mountPath: /www
          name: www-data
      volumes:
      - name: www-data
        emptyDir: {}
```






Deployment with an environment variables:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: env-test
  namespace: k8s-basics
spec:
  selector:
    matchLabels:
      app: env-test
  replicas: 1
  template:
    metadata:
      labels:
        app: env-test
    spec:
      containers:
      - name: shell
        image: registry.access.redhat.com/ubi8/ubi
        command:
        - "bin/bash"
        - "-c"
        - "sleep 10000"
        env:
        - name: SIMPLE_SERVICE_VERSION
          value: "1.0"
```


Config Map:
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: example
  namespace: k8s-basics
data:
  example.property.1: hello
  example.property.2: world
  example.property.file: |-
    property.1=value-1
    property.2=value-2
    property.3=value-3
```


Deployment with a config map:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: env-test
  namespace: k8s-basics
spec:
  selector:
    matchLabels:
      app: env-test
  replicas: 1
  template:
    metadata:
      labels:
        app: env-test
    spec:
      containers:
      - name: shell
        image: registry.access.redhat.com/ubi8/ubi
        command:
        - "bin/bash"
        - "-c"
        - "sleep 10000"
        envFrom:
        - configMapRef:
            name: example
```


Secrets:

https://kubernetesbyexample.com/secrets/

```bash
echo -n "A19fh68B001j" > ./apikey.txt
```
```scrects
kubectl create secret generic apikey --from-file=./apikey.txt
```

```
kubectl describe secrets/apikey
```

Create deployment with secrets:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: env-test
  namespace: k8s-basics
spec:
  selector:
    matchLabels:
      app: env-test
  replicas: 1
  template:
    metadata:
      labels:
        app: env-test
    spec:
      containers:
      - name: shell
        image: registry.access.redhat.com/ubi8/ubi
        command:
        - "bin/bash"
        - "-c"
        - "sleep 10000"
        volumeMounts:
        - name: apikeyvol
          mountPath: "/tmp/apikey"
          readOnly: true
      volumes:
      - name: apikeyvol
        secret:
          secretName: apikey

```

```
oc get pods

NAME                        READY   STATUS    RESTARTS   AGE
env-test-6554b8589d-f72sd   1/1     Running   0          2m58s
```

```
kubectl exec -it env-test-6554b8589d-f72sd -c shell -- bash

bash-4.4$ cat /tmp/apikey/apikey.txt
A19fh68B001j
```
