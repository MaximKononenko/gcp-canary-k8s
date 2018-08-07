
# gcp-canary-k8s

## Setting project

```bash
$ export PROJECT_ID=<canary-lab>
$ gcloud config set project $PROJECT_ID
$ gcloud config set compute/zone us-east1-d
$ sed -i.bak "s#PROJECT_ID#$PROJECT_ID#" app-production.yml
```

___

## Prepare infrastructure

Create cluster:

```bash
$ gcloud container clusters create canary
$ kubectl cluster-info
Kubernetes master is running at https://35.229.96.254
GLBCDefaultBackend is running at https://35.229.96.254/api/v1/namespaces/kube-system/services/default-http-backend:http/proxy
Heapster is running at https://35.229.96.254/api/v1/namespaces/kube-system/services/heapster/proxy
KubeDNS is running at https://35.229.96.254/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy
kubernetes-dashboard is running at https://35.229.96.254/api/v1/namespaces/kube-system/services/https:kubernetes-dashboard:/proxy
Metrics-server is running at https://35.229.96.254/api/v1/namespaces/kube-system/services/https:metrics-server:/proxy

To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.
```

Create prod namespace in GKE:

```bash
$ kubectl create namespace production
```

___

## Build & deploy

Build docker image with app v1.0 and push to GCR:

```bash
$ export app_version=1.0

$ docker build --build-arg version=$app_version -t gcr.io/$PROJECT_ID/app:$app_version .
Sending build context to Docker daemon  16.48MB
Step 1/5 : FROM alpine:latest
 ---> 7328f6f8b418
Step 2/5 : ARG version=1.0
 ---> Running in 33c0893e3b89
 ---> 681c2773c86c
Removing intermediate container 33c0893e3b89
Step 3/5 : COPY source/$version/app /app
 ---> f7e99a4bf59e
Removing intermediate container b53150f679ed
Step 4/5 : EXPOSE 8080
 ---> Running in d5cb1c07e26c
 ---> 1534a637c45e
Removing intermediate container d5cb1c07e26c
Step 5/5 : ENTRYPOINT /app
 ---> Running in 7ed3912c444c
 ---> 904295e0ba03
Removing intermediate container 7ed3912c444c
Successfully built 904295e0ba03
Successfully tagged gcr.io/project-canary-181811/app:1.0

$ gcloud docker -- push gcr.io/$PROJECT_ID/app:$app_version
The push refers to a repository [gcr.io/project-canary-181811/app]
badbaf9deb07: Pushed
5bef08742407: Layer already exists
1.0: digest: sha256:7806df2d49603f13c81c0e392e6ccf4bcb57e9928760740870a26ad664d18e05 size: 739
```

Deploy app v1.0 to k8s:

```bash
$ kubectl --namespace=production apply -f app-production.yml
$ kubectl --namespace=production rollout status deployment/kubeapp-production
Waiting for rollout to finish: 1 of 3 updated replicas are available...
Waiting for rollout to finish: 2 of 3 updated replicas are available...
deployment "kubeapp-production" successfully rolled out
#check status
$ kubectl --namespace=production get pods -o wide
NAME                                 READY     STATUS    RESTARTS   AGE       IP           NODE
kubeapp-production-b9cfb665c-4shn9   1/1       Running   0          19h       10.60.1.15   gke-canary-default-pool-832370c5-n803
kubeapp-production-b9cfb665c-n9d88   1/1       Running   0          19h       10.60.1.13   gke-canary-default-pool-832370c5-n803
kubeapp-production-b9cfb665c-px6np   1/1       Running   0          19h       10.60.1.14   gke-canary-default-pool-832370c5-n803

#checking events
$ kubectl --namespace=production get events -w
```

___

## Expose service using GCE LB

![LB](https://cdn-images-1.medium.com/max/1080/1*QZMJoE5e7sooxAa2KhrI8g.png "GCE LB")

```bash
$ kubectl --namespace=production apply -f app-lb.yml
```

Grab the public IP addresses of the LB using the get command:

```bash
$ kubectl --namespace=production get services
```

We can also extract the IP address of the LB using this one-liner:

```bash
$ export SERVICE_IP=$(kubectl --namespace=production get service/app-lb --output=json | jq -r '.status.loadBalancer.ingress[0].ip')

$ curl http://$SERVICE_IP/
Congratulations! Version 1.0 of your application is running on Kubernetes.

$ curl http://$SERVICE_IP/version
1.0
```

## Deploying Canary to k8s

Build and push Docker image for 2.0:

```bash
$ export app_version=2.0

$ docker build --build-arg version=$app_version -t gcr.io/$PROJECT_ID/app:$app_version .
Sending build context to Docker daemon  16.48MB
Step 1/5 : FROM alpine:latest
 ---> 7328f6f8b418
Step 2/5 : ARG version=1.0
 ---> Using cache
 ---> d486d81470ff
Step 3/5 : COPY source/$version/app /app
 ---> f78c9f9d2e78
Removing intermediate container e18187a82f95
Step 4/5 : EXPOSE 8080
 ---> Running in 564b1fc9a62b
 ---> ea31e9bf6605
Removing intermediate container 564b1fc9a62b
Step 5/5 : ENTRYPOINT /app
 ---> Running in 4fb23e91e8b3
 ---> fcd6a752131f
Removing intermediate container 4fb23e91e8b3
Successfully built fcd6a752131f
Successfully tagged gcr.io/project-canary-181811/app:2.0

$ gcloud docker -- push gcr.io/$PROJECT_ID/app:$app_version
The push refers to a repository [gcr.io/project-canary-181811/app]
a02d26201799: Pushed
5bef08742407: Layer already exists
2.0: digest: sha256:ba1514d99861beb78bf74e434859691173099f30b3b6d316fc791ff51741b3d7 size: 739
```

Release the Canary:

```bash
# Parse PROJECT_ID in k8s config
$ sed -i.bak "s#PROJECT_ID#$PROJECT_ID#" app-canary.yml

$ kubectl --namespace=production apply -f app-canary.yml
deployment "kubeapp-canary" created

$ kubectl --namespace=production get pods
NAME                                  READY     STATUS    RESTARTS   AGE
kubeapp-canary-469258524-2qjk0        1/1       Running   0          32s
kubeapp-production-1443420586-58njk   1/1       Running   0          1h
kubeapp-production-1443420586-mc5g2   1/1       Running   0          1h
kubeapp-production-1443420586-vf9x6   1/1       Running   0          1h
```

Simulate users hitting the app:

```bash
$ for i in `seq 1 10`; do curl http://$SERVICE_IP/version; sleep 1;  done
1.0
1.0
1.0
2.0
1.0
2.0
1.0
2.0
1.0
1.0
```

Once 2.0 is tested and approved, we can proceed by switching the Docker image used in app-production.yml to 2.0 and then simply re-apply the configuration. Another route would be to save the image used directly to the k8s API:

```bash
$ kubectl --namespace=production set image deployment/kubeapp-production kubeapp=gcr.io/$PROJECT_ID/app:2.0
```

This will instruct k8s to launch new pods while terminating the old ones. By this point, you don’t really need the Canary deployment. You can go ahead and delete it by executing **kubectl --namespace=production delete deployment/kubeapp-canary**

___

## Using Ingress resources to access Canary Deployments

Ingress resources are provided in k8s by an Ingress Controller. Those controllers abstract away the inner workings of the backend software used for implementing advanced load balancing (could even be nginx, GCE LB, HAProxy, etc) and let us focus on the routing rules.

To be able to access the Canary release on a separate subdomain, we need to add some Ingress routes that point to the Canary Service. The approach we’re using here is name-based virtual hosting, where the Ingress resource will check the host header and judge which service it should send the traffic to.

```
foo.bar        --|                 |-> kubeapp-production-service 80
                 |   Ingress IP    |
canary.foo.bar --|                 |-> kubeapp-canary-service     80
```

```bash
$ kubectl --namespace=production apply -f app-ingress-canary.yml
ingress "app-ingress" created

# If the backends report HEALTHY, we can start sending requests for testing the endpoints
$ kubectl --namespace=production describe ing
Name:			app-ingress
Namespace:		production
Address:		35.190.60.217
Default backend:	kubeapp-production-service:80 (10.44.1.10:8080,10.44.1.8:8080,10.44.1.9:8080)
Rules:
  Host			Path	Backends
  ----			----	--------
  canary.foo.bar
    			 	kubeapp-canary-service:80 (10.44.1.11:8080)
  foo.bar
    			 	kubeapp-production-service:80 (10.44.1.10:8080,10.44.1.8:8080,10.44.1.9:8080)
Annotations:
  forwarding-rule:	k8s-fw-production-app-ingress--c01b1884e664276f
  target-proxy:		k8s-tp-production-app-ingress--c01b1884e664276f
  url-map:		k8s-um-production-app-ingress--c01b1884e664276f
  backends:		{"k8s-be-31930--c01b1884e664276f":"HEALTHY","k8s-be-32638--c01b1884e664276f":"HEALTHY"}
Events:
  FirstSeen	LastSeen	Count	From			SubObjectPath	Type		Reason	Message
  ---------	--------	-----	----			-------------	--------	------	-------
  15m		15m		1	loadbalancer-controller			Normal		ADD	production/app-ingress
  13m		13m		1	loadbalancer-controller			Normal		CREATE	ip: 35.190.60.217
  14m		2m		5	loadbalancer-controller			Normal		Service	default backend set to kubeapp-production-service:31930

# Set resolver to point to Ingress resource IP
$ export INGRESS_IP=$(kubectl --namespace=production get ing/app-ingress --output=json | jq -r '.status.loadBalancer.ingress[0].ip')
$ echo "$INGRESS_IP foo.bar canary.foo.bar" | sudo tee -a /etc/hosts

$ curl http://foo.bar/
Congratulations! Version 1.0 of your application is running on Kubernetes.

$ curl http://foo.bar/version
1.0

$ curl http://canary.foo.bar/
Congratulations! Version 2.0 of your application is running on Kubernetes.

$ curl http://canary.foo.bar/version
2.0
```

To recreate production namcpace use:

```bash
kubectl delete namespace production && kubectl create namespace production
```