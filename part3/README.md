pod lifecycle :-
pending = waiting for node assigned
containercoreating = pulling image, starting it, attaching network
Running
ERROR
crashloopbackoff = process dying too many times
succeeded = when work is cpmpleted, jobs etc.


$ kubectl get node node01 -o jsonpath='{.spec.podCIDR}'  (to check the pod cidr range)
$ kubectl get pods -owide


### simple pod
```
apiVersion: v1
kind: Pod
metadata:
  name: example-pod
  labels:
    purpose: demonstrate-args
spec:
  containers:
  - name: example-container
    image: ubuntu
    command: ["/bin/echo", "Hello"]  
    args: ["Welcome", "to", "Kubesimplify"]
```
 ### CM
 ```
apiVersion: v1
kind: ConfigMap
metadata:
  name: example-configmap-folded
data:
  mymessage: >
    Hello, this is a folded
    multi-line message.
 ```   

### Simple Pod
```
apiVersion: batch/v1
kind: Job
metadata:
  name: nginx-job
spec:
  template:
    spec:
      containers:
      - name: nginx
        image: nginx
        command: ["sleep", "5"]  
      restartPolicy: OnFailure  
  backoffLimit: 4 

--------------------------------------------Init Container & Sidecar Container-----------------------------------------------------

Init Container :- 

In Kubernetes, an init container is a special type of container that runs before the main application containers in a Pod. 
Init containers are useful for performing initialization tasks that need to be completed before the application starts. Here are 
some key points about init containers:

use case:- Checking the availability of services or resources that the main application depends on (like waiting for a database 
to become ready).
use case:- Downloading configuration files or secrets from an external source like github.


ubuntu@ip-172-31-47-215:~/pods$ cat initcontainer.yml
kind: Pod
apiVersion: v1
metadata:
  name: init-test

spec:
  initContainers:
  - name: init-container
    image: busybox:latest
    command: ["sh","-c", "echo 'Initalization started ...'; sleep 10; echo 'Initization completed.'"]

  containers:
  - name: main-container
    image: busybox:latest
    command: ["sh","-c", "echo 'Main container started'"]

$ kubectl apply -f initcontainer.yml
$ kubectl get pods
$ kubectl logs init-test -c init-container
$ kubectl logs init-test -c main-container


Sidecar Container :-

in sidecar container if we want to parrelelly run a container with main container so we create sidecar container.

One container serves as the main application, while another provides auxiliary functions like logging, monitoring, or proxying requests

in below lab you can see main container will produce logs and sidecar container will display the logs

ubuntu@ip-172-31-47-215:~/pods$ cat sidecar-container.yml
kind: Pod
apiVersion: v1
metadata:
  name: sidecar-test

spec:

  volumes:
  - name: shared-logs
    emptyDir: {}

  containers:
  # produce logs
  - name: main-container
    image: busybox
    command: ["sh","-c", "while true; do echo 'Hello Dosto' >> /var/log/app.log; sleep 5; done"]
    volumeMounts:
    - name: shared-logs
      mountPath:  /var/log/

  # display logs
  - name: sidecar-container
    image: busybox
    command: ["sh","-c", "tail -f /var/log/app.log"]
    volumeMounts:
    - name: shared-logs
      mountPath:  /var/log/

$ kubectl apply -f sidecar-container.yml
$ kubectl logs sidecar-test -c main-container
$ kubectl logs sidecar-test -c sidecar-container


## init container
```
apiVersion: v1
kind: Pod
metadata:
  name: bootcamp-pod
spec:
  volumes:
  - name: shared-data
    emptyDir: {}   # multiple containers can access the same directory
  initContainers:
  - name: bootcamp-init
    image: busybox
    command: ['sh', '-c', 'wget -O /usr/share/data/index.html http://kubesimplify.com']
    volumeMounts:
    - name: shared-data
      mountPath: /usr/share/data
  containers:
  - name: nginx
    image: nginx
    volumeMounts:
    - name: shared-data
      mountPath: /usr/share/nginx/html
```

$ kubectl get pod/bootcamp-pod -owide
$ curl 192.168.1.6       (whatever html code we have downloaded, we can check it)

----------------------------------------------------------------------------------
here in below example our main container is dependent on 2 services so first will create init container pod and check if its running
so it will give error, then will create both required services and check our main container will be running.

### multiple init container
```
apiVersion: v1
kind: Pod
metadata:
  name: init-demo-2
spec:
  initContainers:
  - name: check-db-service
    image: busybox
    command: ['sh', '-c', 'until nslookup db.default.svc.cluster.local; do echo waiting for db service; sleep 2; done;']
  - name: check-myservice
    image: busybox
    command: ['sh', '-c', 'until nslookup myservice.default.svc.cluster.local; do echo waiting for db service; sleep 2; done;']
  containers:
  - name: main-container
    image: busybox
    command: ['sleep', '3600']

$ kubectl apply -f multi.yml
$ kubectl get pod/init-demo-2


apiVersion: v1
kind: Service
metadata:
  name: db
spec:
  selector:
    app: demo1
  ports:
  - protocol: TCP
    port: 3306
    targetPort: 3306
---
apiVersion: v1
kind: Service
metadata:
  name: myservice
spec:
  selector:
    app: demo2
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80

$ kubectl apply -f svc.yml
$ kubectl get pod/init-demo-2

--------------------------------------------------------------

### Multi container pod

apiVersion: v1
kind: Pod
metadata:
  name: multi-container-pod
spec:
  volumes:
  - name: shared-data
    emptyDir: {}
  initContainers:
  - name: meminfo-container
    image: alpine
    restartPolicy: Always
    command: ['sh', '-c', 'sleep 5; while true; do cat /proc/meminfo > /usr/share/data/index.html; sleep 10; done;']
    volumeMounts:
    - name: shared-data
      mountPath: /usr/share/data
  containers:
  - name: nginx-container
    image: nginx
    volumeMounts:
    - name: shared-data
      mountPath: /usr/share/nginx/html
      
