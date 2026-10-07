# Helm Homework — Session 15

Bibek Jyoti Charah — 24bcs10112 (GitHub: SammyUrfen)

I ran every command on Fedora with minikube v1.38.1 (Docker driver, profile `s15`, 2 CPUs, 2200 MB),
Kubernetes v1.35.1, kubectl v1.37.0 and Helm v4.2.2. All `kubectl` and `helm` commands ran against the
`s15` profile. Output is pasted as printed. Long output is cut with `...`. The full transcripts are in
[`outputs/`](outputs/).

## Repository layout

| Path | What it is |
| :--- | :--- |
| `hello-chart/` | My chart: nginx serves one message from a ConfigMap. Used for Task 1 and Task 2. |
| `task1/demo-chart/` | The skeleton from `helm create`, unchanged. |
| `task2/values-v3.yaml` | The values file for the second upgrade in Task 2. |
| `mini-project/notes-chart/` | The course Notes chart (Task 3), copied unchanged from the course repo. |
| `outputs/` | Full terminal transcripts of every run. |

## Coverage

| Homework item | Section |
| :--- | :--- |
| Task 1: `helm create` | [1.1](#11-helm-create) |
| Task 1: `helm lint`, `helm template` | [1.2](#12-helm-lint-and-helm-template-on-my-chart) |
| Task 1: `helm install`, `list`, `status` | [1.3](#13-helm-install-list-status) |
| Task 1: `helm get values/manifest/notes/all` | [1.4](#14-helm-get) |
| Task 1: `helm upgrade`, `history`, `rollback` | [Task 2](#task-2--rollback-workflow) |
| Task 1: `helm uninstall` | [1.5](#15-helm-uninstall) |
| Task 1: `helm repo add/list/update/remove` | [1.6](#16-helm-repo) |
| Task 1: `helm search repo`, `helm search hub` | [1.7](#17-helm-search) |
| Task 2: install, upgrade, verify, upgrade, verify, rollback, verify | [Task 2](#task-2--rollback-workflow) |
| Task 3: Notes app mini project | [Task 3](#task-3--mini-project-notes-app) |
| Helm v3 to v4 differences | [Findings](#findings) |

## My chart: `hello-chart`

The chart has a ConfigMap, a Deployment and a Service. The ConfigMap holds `index.html`, and nginx
serves it. The page shows the message, the image tag and the chart version, so `curl` proves which
values the running pods use. The Deployment has a `checksum/html` annotation. It holds a hash of the
ConfigMap, so a new message also restarts the pods. Without it, a ConfigMap change alone does not
restart them.

`hello-chart/values.yaml`:

```yaml
replicaCount: 1

image:
  repository: nginx
  tag: "1.27"

# Text served at / . Changing it rolls the pods (see checksum annotation).
message: "Hello from revision one"

service:
  port: 80

resources:
  requests: {cpu: 10m, memory: 32Mi}
  limits: {cpu: 100m, memory: 64Mi}
```

---

## Task 1 — Helm commands

### 1.1 `helm create`

`helm create` writes a new chart skeleton: Chart.yaml, values.yaml, templates and a test hook.

```text
$ helm version
version.BuildInfo{Version:"v4.2.2", GitCommit:"b05881cf967a5a09e19866799d0edfd40675803a", GitTreeState:"clean", GoVersion:"go1.26.4-X:nodwarf5", KubeClientVersion:"v1.36"}

$ helm create demo-chart
Creating demo-chart

$ ls demo-chart demo-chart/templates
demo-chart:
charts
Chart.yaml
templates
values.yaml

demo-chart/templates:
deployment.yaml
_helpers.tpl
hpa.yaml
httproute.yaml
ingress.yaml
NOTES.txt
serviceaccount.yaml
service.yaml
tests
```

The Helm v4 skeleton includes `httproute.yaml` (Gateway API). The Helm v3 skeleton does not.

### 1.2 `helm lint` and `helm template` on my chart

`helm lint` checks the chart for errors. `helm template` renders the YAML locally. It needs no cluster.

```text
$ helm lint hello-chart
==> Linting hello-chart
[INFO] Chart.yaml: icon is recommended

1 chart(s) linted, 0 chart(s) failed

$ helm template hello hello-chart
---
# Source: hello-chart/templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: hello-html
  labels:
    app: hello
data:
  index.html: |
    Hello from revision one (image nginx:1.27, chart 0.1.0)

---
# Source: hello-chart/templates/service.yaml
apiVersion: v1
kind: Service
metadata:
  name: hello
  labels:
    app: hello
spec:
  type: ClusterIP
  selector:
    app: hello
  ports:
    - port: 80
      targetPort: 80

---
# Source: hello-chart/templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello
  labels:
    app: hello
    chart: hello-chart-0.1.0
spec:
  replicas: 1
  selector:
    matchLabels:
      app: hello
  template:
    metadata:
      labels:
        app: hello
      annotations:
        # A ConfigMap change alone does not restart pods; this hash makes it do so.
        checksum/html: fc4dd4a1c526b056aacee686fa1cfb8b0849388ed8d59519cbdd83f059df651a
    spec:
      containers:
        - name: web
          image: "nginx:1.27"
          ports:
            - containerPort: 80
          env:
            - name: MESSAGE
              value: "Hello from revision one"
          readinessProbe:
            httpGet: {path: /, port: 80}
          resources:
            limits:
              cpu: 100m
              memory: 64Mi
            requests:
              cpu: 10m
              memory: 32Mi
          volumeMounts:
            - name: html
              mountPath: /usr/share/nginx/html
      volumes:
        - name: html
          configMap:
            name: hello-html
```

### 1.3 `helm install`, `list`, `status`

`helm install` creates a release (revision 1). `helm list` shows the releases in the namespace.
`helm status` shows the state of one release.

```text
$ helm install hello hello-chart --wait --timeout 3m
NAME: hello
LAST DEPLOYED: Wed Oct  7 20:48:28 2026
NAMESPACE: default
STATUS: deployed
REVISION: 1
DESCRIPTION: Install complete
TEST SUITE: None
NOTES:
hello is installed: 1 replica(s) of nginx:1.27.
Test it:
  kubectl port-forward svc/hello 41580:80
  curl http://localhost:41580/

$ helm list
NAME 	NAMESPACE	REVISION	UPDATED                                	STATUS  	CHART            	APP VERSION
hello	default  	1       	2026-10-07 20:48:28.748231605 +0530 IST	deployed	hello-chart-0.1.0	1.27       

$ helm status hello
NAME: hello
LAST DEPLOYED: Wed Oct  7 20:48:28 2026
NAMESPACE: default
STATUS: deployed
REVISION: 1
DESCRIPTION: Install complete
RESOURCES:
==> v1/ConfigMap
NAME         DATA   AGE
hello-html   1      22s

==> v1/Service
NAME    TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)   AGE
hello   ClusterIP   10.104.174.247   <none>        80/TCP    22s

==> v1/Deployment
NAME    READY   UP-TO-DATE   AVAILABLE   AGE
hello   1/1     1            1           21s

==> v1/Pod(related)
NAME                     READY   STATUS    RESTARTS   AGE
hello-686d4f8874-j9lbk   1/1     Running   0          21s


TEST SUITE: None
NOTES:
...
```

In Helm v4, `helm status` lists the release resources by default. Helm v3 needed `--show-resources`.

### 1.4 `helm get`

`helm get` reads what Helm stored for a release. `values` shows the values you supplied. `--all`
adds the chart defaults. `manifest` shows the YAML that Helm sent. `notes` shows NOTES.txt.
`all` shows everything at once.

```text
$ helm get values hello
USER-SUPPLIED VALUES:
null

$ helm get values hello --all
COMPUTED VALUES:
image:
  repository: nginx
  tag: "1.27"
message: Hello from revision one
replicaCount: 1
resources:
  limits:
    cpu: 100m
    memory: 64Mi
  requests:
    cpu: 10m
    memory: 32Mi
service:
  port: 80

$ helm get manifest hello
---
# Source: hello-chart/templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: hello-html
...

$ helm get notes hello
NOTES:
hello is installed: 1 replica(s) of nginx:1.27.
Test it:
  kubectl port-forward svc/hello 41580:80
  curl http://localhost:41580/

$ helm get all hello
NAME: hello
LAST DEPLOYED: Wed Oct  7 20:48:28 2026
NAMESPACE: default
STATUS: deployed
REVISION: 1
CHART: hello-chart
VERSION: 0.1.0
APP_VERSION: 1.27
DESCRIPTION: Install complete
TEST SUITE: None
USER-SUPPLIED VALUES:
null

COMPUTED VALUES:
...
HOOKS:
MANIFEST:
...
NOTES:
...
```

`null` means I passed no `--set` and no `-f`. Full output: [`outputs/task1-create-install-get.txt`](outputs/task1-create-install-get.txt).

### 1.5 `helm uninstall`

`helm uninstall` deletes the release resources and the release history.

```text
$ helm uninstall hello --wait
release "hello" uninstalled

$ helm list
NAME	NAMESPACE	REVISION	UPDATED	STATUS	CHART	APP VERSION

$ kubectl get all -l app=hello
NAME                         READY   STATUS        RESTARTS   AGE
pod/hello-69d57bccdc-9cb5s   1/1     Terminating   0          17s
pod/hello-69d57bccdc-g8lqh   1/1     Terminating   0          14s
pod/hello-69d57bccdc-vc7fj   1/1     Terminating   0          21s
```

The pods were still in `Terminating` even with `--wait`. Helm waits for the Deployment to go. The
pods go a few seconds later.

### 1.6 `helm repo`

A repo is an index of packaged charts at a URL. My machine already had repos in its Helm config.
For a clean demo, I pointed `HELM_CONFIG_HOME`, `HELM_CACHE_HOME` and `HELM_DATA_HOME` at an empty
temporary directory.

```text
$ helm repo list
no repositories to show

$ helm repo add bitnami https://charts.bitnami.com/bitnami
"bitnami" has been added to your repositories

$ helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
"prometheus-community" has been added to your repositories

$ helm repo list
NAME                	URL                                               
bitnami             	https://charts.bitnami.com/bitnami                
prometheus-community	https://prometheus-community.github.io/helm-charts

$ helm repo update
Hang tight while we grab the latest from your chart repositories...
...Successfully got an update from the "prometheus-community" chart repository
...Successfully got an update from the "bitnami" chart repository
Update Complete. ⎈Happy Helming!⎈

$ helm repo remove prometheus-community
"prometheus-community" has been removed from your repositories

$ helm repo list
NAME   	URL                               
bitnami	https://charts.bitnami.com/bitnami
```

`repo add` saves the URL. `repo update` downloads the latest index. `repo remove` deletes the entry.

### 1.7 `helm search`

`helm search repo` searches the repos added on this machine. `helm search hub` searches Artifact
Hub on the internet.

```text
$ helm search repo bitnami/nginx
NAME                            	CHART VERSION	APP VERSION	DESCRIPTION                                       
bitnami/nginx                   	25.2.1       	1.31.6     	NGINX Open Source is a web server that can be a...
bitnami/nginx-ingress-controller	12.0.7       	1.13.1     	NGINX Ingress Controller is an Ingress controll...
bitnami/nginx-intel             	2.1.15       	0.4.9      	DEPRECATED NGINX Open Source for Intel is a lig...

$ helm search repo nginx --versions | head -5
NAME                                          	CHART VERSION	APP VERSION	DESCRIPTION                                       
bitnami/nginx                                 	25.2.1       	1.31.6     	NGINX Open Source is a web server that can be a...
bitnami/nginx                                 	25.2.0       	1.31.6     	NGINX Open Source is a web server that can be a...
bitnami/nginx                                 	25.1.15      	1.31.6     	NGINX Open Source is a web server that can be a...
bitnami/nginx                                 	25.1.14      	1.31.6     	NGINX Open Source is a web server that can be a...

$ helm search hub nginx --max-col-width 50 | head -6
URL                                               	CHART VERSION  	APP VERSION                                     	DESCRIPTION                                       
https://artifacthub.io/packages/helm/cloudpirat...	0.16.12        	1.31.6                                          	Nginx is a high-performance HTTP server and rev...
https://artifacthub.io/packages/helm/quench-ngi...	0.0.15         	1.30.5                                          	High-performance web server, reverse proxy, and...
https://artifacthub.io/packages/helm/krakazyabr...	1.0.0          	1.19.0                                          	Nginx Helm chart for Kubernetes                   
https://artifacthub.io/packages/helm/dhinesh/nginx	25.2.1         	1.31.6                                          	NGINX Open Source is a web server that can be a...
https://artifacthub.io/packages/helm/bitnami/nginx	25.2.1         	1.31.6                                          	NGINX Open Source is a web server that can be a...
```

`CHART VERSION` is the version of the package. `APP VERSION` is the version of nginx inside it.

---

## Task 2 — Rollback workflow

Plan:

| Stage | Command | Replicas | Image | Message |
| :--- | :--- | :--- | :--- | :--- |
| Install (rev 1) | `helm install` | 1 | nginx:1.27 | Hello from revision one |
| Upgrade (rev 2) | `--set` | 3 | nginx:1.28 | Hello from revision two |
| Upgrade (rev 3) | `-f task2/values-v3.yaml` | 2 | nginx:1.29 | Hello from revision three (values file) |
| Rollback (rev 4) | `helm rollback hello` | 3 | nginx:1.28 | Hello from revision two |
| Rollback (rev 5) | `helm rollback hello 1` | 1 | nginx:1.27 | Hello from revision one |

Each "verify" step runs the same checks: `kubectl rollout status`, the Deployment image, the image
of every pod, `printenv MESSAGE` inside every pod, and `curl` through
`kubectl port-forward svc/hello 41580:80`. Full transcript: [`outputs/task2-rollback-workflow.txt`](outputs/task2-rollback-workflow.txt).

### Stage 1 — install and verify

```text
$ helm install hello hello-chart --wait --timeout 3m
NAME: hello
LAST DEPLOYED: Wed Oct  7 20:51:03 2026
NAMESPACE: default
STATUS: deployed
REVISION: 1
DESCRIPTION: Install complete
...

$ helm history hello
REVISION	UPDATED                 	STATUS  	CHART            	APP VERSION	DESCRIPTION     
1       	Wed Oct  7 20:51:03 2026	deployed	hello-chart-0.1.0	1.27       	Install complete

$ kubectl get deploy hello -o wide
NAME    READY   UP-TO-DATE   AVAILABLE   AGE   CONTAINERS   IMAGES       SELECTOR
hello   1/1     1            1           3s    web          nginx:1.27   app=hello

$ kubectl get pods -l app=hello -o 'custom-columns=POD:.metadata.name,STATUS:.status.phase,IMAGE:.spec.containers[0].image'
POD                      STATUS    IMAGE
hello-686d4f8874-rbfsk   Running   nginx:1.27

$ kubectl exec pod/hello-686d4f8874-rbfsk -- printenv MESSAGE
Hello from revision one

$ curl -s http://localhost:41580/
Hello from revision one (image nginx:1.27, chart 0.1.0)
```

### Stage 2 — first upgrade (`--set`) and verify

```text
$ helm upgrade hello hello-chart --set replicaCount=3 --set image.tag=1.28 --set message='Hello from revision two' --wait --timeout 3m
Release "hello" has been upgraded. Happy Helming!
NAME: hello
LAST DEPLOYED: Wed Oct  7 20:51:10 2026
NAMESPACE: default
STATUS: deployed
REVISION: 2
DESCRIPTION: Upgrade complete
...

$ helm history hello
REVISION	UPDATED                 	STATUS    	CHART            	APP VERSION	DESCRIPTION     
1       	Wed Oct  7 20:51:03 2026	superseded	hello-chart-0.1.0	1.27       	Install complete
2       	Wed Oct  7 20:51:10 2026	deployed  	hello-chart-0.1.0	1.27       	Upgrade complete

$ helm get values hello
USER-SUPPLIED VALUES:
image:
  tag: "1.28"
message: Hello from revision two
replicaCount: 3

$ kubectl get deploy hello -o wide
NAME    READY   UP-TO-DATE   AVAILABLE   AGE   CONTAINERS   IMAGES       SELECTOR
hello   3/3     3            3           19s   web          nginx:1.28   app=hello

$ kubectl get pods -l app=hello -o 'custom-columns=POD:.metadata.name,STATUS:.status.phase,IMAGE:.spec.containers[0].image'
POD                      STATUS    IMAGE
hello-69d57bccdc-6xwvv   Running   nginx:1.28
hello-69d57bccdc-lpgdk   Running   nginx:1.28
hello-69d57bccdc-txl9g   Running   nginx:1.28

$ kubectl exec pod/hello-69d57bccdc-6xwvv -- printenv MESSAGE
Hello from revision two

$ kubectl exec pod/hello-69d57bccdc-lpgdk -- printenv MESSAGE
Hello from revision two

$ kubectl exec pod/hello-69d57bccdc-txl9g -- printenv MESSAGE
Hello from revision two

$ curl -s http://localhost:41580/
Hello from revision two (image nginx:1.28, chart 0.1.0)
```

### Stage 3 — second upgrade (values file) and verify

```text
$ cat task2/values-v3.yaml
replicaCount: 2
image:
  tag: "1.29"
message: "Hello from revision three (values file)"

$ helm upgrade hello hello-chart -f task2/values-v3.yaml --wait --timeout 3m
Release "hello" has been upgraded. Happy Helming!
NAME: hello
LAST DEPLOYED: Wed Oct  7 20:51:26 2026
NAMESPACE: default
STATUS: deployed
REVISION: 3
DESCRIPTION: Upgrade complete
...

$ helm history hello
REVISION	UPDATED                 	STATUS    	CHART            	APP VERSION	DESCRIPTION     
1       	Wed Oct  7 20:51:03 2026	superseded	hello-chart-0.1.0	1.27       	Install complete
2       	Wed Oct  7 20:51:10 2026	superseded	hello-chart-0.1.0	1.27       	Upgrade complete
3       	Wed Oct  7 20:51:26 2026	deployed  	hello-chart-0.1.0	1.27       	Upgrade complete

$ kubectl get deploy hello -o wide
NAME    READY   UP-TO-DATE   AVAILABLE   AGE   CONTAINERS   IMAGES       SELECTOR
hello   2/2     2            2           32s   web          nginx:1.29   app=hello

$ kubectl get pods -l app=hello -o 'custom-columns=POD:.metadata.name,STATUS:.status.phase,IMAGE:.spec.containers[0].image'
POD                      STATUS    IMAGE
hello-75fc9bbb47-m2hnm   Running   nginx:1.29
hello-75fc9bbb47-pk2nj   Running   nginx:1.29

$ kubectl exec pod/hello-75fc9bbb47-m2hnm -- printenv MESSAGE
Hello from revision three (values file)

$ kubectl exec pod/hello-75fc9bbb47-pk2nj -- printenv MESSAGE
Hello from revision three (values file)

$ curl -s http://localhost:41580/
Hello from revision three (values file) (image nginx:1.29, chart 0.1.0)
```

### Stage 4 — rollback and verify

`helm rollback hello` with no number goes to the previous revision (2). It does not delete revision 3.
It writes a new revision 4 with a copy of the revision 2 values.

```text
$ helm rollback hello --wait --timeout 3m
Rollback was a success! Happy Helming!

$ helm history hello
REVISION	UPDATED                 	STATUS    	CHART            	APP VERSION	DESCRIPTION     
1       	Wed Oct  7 20:51:03 2026	superseded	hello-chart-0.1.0	1.27       	Install complete
2       	Wed Oct  7 20:51:10 2026	superseded	hello-chart-0.1.0	1.27       	Upgrade complete
3       	Wed Oct  7 20:51:26 2026	superseded	hello-chart-0.1.0	1.27       	Upgrade complete
4       	Wed Oct  7 20:51:39 2026	deployed  	hello-chart-0.1.0	1.27       	Rollback to 2   

$ helm get values hello
USER-SUPPLIED VALUES:
image:
  tag: "1.28"
message: Hello from revision two
replicaCount: 3

$ kubectl get deploy hello -o wide
NAME    READY   UP-TO-DATE   AVAILABLE   AGE   CONTAINERS   IMAGES       SELECTOR
hello   3/3     3            3           50s   web          nginx:1.28   app=hello

$ kubectl get pods -l app=hello -o 'custom-columns=POD:.metadata.name,STATUS:.status.phase,IMAGE:.spec.containers[0].image'
POD                      STATUS    IMAGE
hello-69d57bccdc-7kt22   Running   nginx:1.28
hello-69d57bccdc-lnqkz   Running   nginx:1.28
hello-69d57bccdc-snz9c   Running   nginx:1.28

$ kubectl exec pod/hello-69d57bccdc-7kt22 -- printenv MESSAGE
Hello from revision two

$ kubectl exec pod/hello-69d57bccdc-lnqkz -- printenv MESSAGE
Hello from revision two

$ kubectl exec pod/hello-69d57bccdc-snz9c -- printenv MESSAGE
Hello from revision two

$ curl -s http://localhost:41580/
Hello from revision two (image nginx:1.28, chart 0.1.0)

$ kubectl get rs -l app=hello
NAME               DESIRED   CURRENT   READY   AGE
hello-686d4f8874   0         0         0       54s
hello-69d57bccdc   3         3         3       47s
hello-75fc9bbb47   0         0         0       31s
```

Proof that the rollback restored the old values in the running pods:

- All three pods print `Hello from revision two` and run `nginx:1.28`, the same as revision 2.
- The pods use the ReplicaSet hash `69d57bccdc`, the same hash as in Stage 2. The pod template is
  identical to revision 2, so Kubernetes scaled the old ReplicaSet back up.

### Stage 5 — rollback to revision 1 and verify

```text
$ helm rollback hello 1 --wait --timeout 3m
Rollback was a success! Happy Helming!

$ helm history hello
REVISION	UPDATED                 	STATUS    	CHART            	APP VERSION	DESCRIPTION     
1       	Wed Oct  7 20:51:03 2026	superseded	hello-chart-0.1.0	1.27       	Install complete
2       	Wed Oct  7 20:51:10 2026	superseded	hello-chart-0.1.0	1.27       	Upgrade complete
3       	Wed Oct  7 20:51:26 2026	superseded	hello-chart-0.1.0	1.27       	Upgrade complete
4       	Wed Oct  7 20:51:39 2026	superseded	hello-chart-0.1.0	1.27       	Rollback to 2   
5       	Wed Oct  7 20:51:57 2026	deployed  	hello-chart-0.1.0	1.27       	Rollback to 1   

$ helm get values hello
USER-SUPPLIED VALUES:
null

$ kubectl get deploy hello -o wide
NAME    READY   UP-TO-DATE   AVAILABLE   AGE   CONTAINERS   IMAGES       SELECTOR
hello   1/1     1            1           61s   web          nginx:1.27   app=hello

$ kubectl get pods -l app=hello -o 'custom-columns=POD:.metadata.name,STATUS:.status.phase,IMAGE:.spec.containers[0].image'
POD                      STATUS    IMAGE
hello-686d4f8874-5mjvj   Running   nginx:1.27

$ kubectl exec pod/hello-686d4f8874-5mjvj -- printenv MESSAGE
Hello from revision one

$ curl -s http://localhost:41580/
Hello from revision one (image nginx:1.27, chart 0.1.0)
```

### Extra — automatic rollback on a failed upgrade

A failed upgrade with `--rollback-on-failure` (the Helm v4 name of `--atomic`) rolls back by itself:

```text
$ helm upgrade hello hello-chart --set image.tag=doesnotexist --atomic --timeout 45s
Flag --atomic has been deprecated, use --rollback-on-failure instead
level=WARN msg="upgrade failed" name=hello error="resource Deployment/default/hello not ready. status: InProgress, message: Pending termination: 1\ncontext deadline exceeded"
Error: UPGRADE FAILED: release hello failed, and has been rolled back due to rollback-on-failure being set: resource Deployment/default/hello not ready. status: InProgress, message: Pending termination: 1
context deadline exceeded

$ helm upgrade hello hello-chart --set image.tag=doesnotexist --rollback-on-failure --timeout 45s
...
Error: UPGRADE FAILED: release hello failed, and has been rolled back due to rollback-on-failure being set: ...

$ helm history hello
REVISION	UPDATED                 	STATUS    	CHART            	APP VERSION	DESCRIPTION
...
5       	Wed Oct  7 20:51:57 2026	superseded	hello-chart-0.1.0	1.27       	Rollback to 1
6       	Wed Oct  7 20:52:35 2026	failed    	hello-chart-0.1.0	1.27       	Upgrade "hello" failed: resource Deployment/default/hello not ready. status: InProgress, message: Pending termination...
7       	Wed Oct  7 20:53:21 2026	superseded	hello-chart-0.1.0	1.27       	Rollback to 5
8       	Wed Oct  7 20:53:21 2026	failed    	hello-chart-0.1.0	1.27       	Upgrade "hello" failed: resource Deployment/default/hello not ready. status: InProgress, message: Pending termination...
9       	Wed Oct  7 20:54:07 2026	deployed  	hello-chart-0.1.0	1.27       	Rollback to 7

$ kubectl get pods -l app=hello
NAME                     READY   STATUS        RESTARTS   AGE
hello-5944647764-c5459   0/1     Terminating   0          46s
hello-686d4f8874-5mjvj   1/1     Running       0          2m11s
```

The old pod `5mjvj` from revision 1 stayed `Running` the whole time. The broken pod is deleted.

---

## Task 3 — Mini project: Notes app

The chart is the course `notes-chart`, copied unchanged into [`mini-project/notes-chart/`](mini-project/notes-chart/).
It has a ConfigMap (`APP_NAME`, `ENVIRONMENT`), a Deployment that loads it with `envFrom`, and a
NodePort Service on 30090. `values.yaml` is development (1 replica, nginx:1.24). `values-prod.yaml`
is production (3 replicas, nginx:1.25). Full transcript: [`outputs/mini-project.txt`](outputs/mini-project.txt).

### Lint and render (steps 8–9)

```text
$ helm lint notes-chart
==> Linting notes-chart
[INFO] Chart.yaml: icon is recommended

1 chart(s) linted, 0 chart(s) failed

$ helm template notes-dev notes-chart
---
# Source: notes-chart/templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: notes-dev-config
data:
  APP_NAME: "notes-app"
  ENVIRONMENT: "development"
...
        - name: notes
          image: "nginx:1.24"
...
```

### Install development (step 10)

```text
$ helm install notes-dev notes-chart --wait --timeout 3m
NAME: notes-dev
LAST DEPLOYED: Wed Oct  7 21:32:30 2026
NAMESPACE: default
STATUS: deployed
REVISION: 1
DESCRIPTION: Install complete
TEST SUITE: None

$ kubectl get pods
NAME                                READY   STATUS    RESTARTS   AGE
notes-dev-deploy-578b55fc88-9gpfk   1/1     Running   0          2s

$ kubectl get services
NAME            TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
kubernetes      ClusterIP   10.96.0.1       <none>        443/TCP        44m
notes-dev-svc   NodePort    10.108.41.126   <none>        80:30090/TCP   2s

$ kubectl get configmaps
NAME               DATA   AGE
kube-root-ca.crt   1      44m
notes-dev-config   2      2s

$ kubectl get configmap notes-dev-config -o jsonpath='{.data}'
{"APP_NAME":"notes-app","ENVIRONMENT":"development"}

$ kubectl exec deploy/notes-dev-deploy -- printenv ENVIRONMENT APP_NAME
development
notes-app

$ curl -s http://$(minikube -p s15 ip):30090/ | grep -o '<title>.*</title>'
<title>Welcome to nginx!</title>
```

### Upgrade to production values (steps 11–12)

```text
$ helm upgrade notes-dev notes-chart -f notes-chart/values-prod.yaml --wait --timeout 3m
Release "notes-dev" has been upgraded. Happy Helming!
NAME: notes-dev
LAST DEPLOYED: Wed Oct  7 21:32:33 2026
NAMESPACE: default
STATUS: deployed
REVISION: 2
DESCRIPTION: Upgrade complete
TEST SUITE: None

$ kubectl get pods -o 'custom-columns=POD:.metadata.name,STATUS:.status.phase,IMAGE:.spec.containers[0].image'
POD                                 STATUS    IMAGE
notes-dev-deploy-57c655ccc5-2hvdh   Running   nginx:1.25
notes-dev-deploy-57c655ccc5-jjl2l   Running   nginx:1.25
notes-dev-deploy-57c655ccc5-xqd2g   Running   nginx:1.25

$ kubectl get configmap notes-dev-config -o jsonpath='{.data}'
{"APP_NAME":"notes-app","ENVIRONMENT":"production"}

$ helm history notes-dev
REVISION	UPDATED                 	STATUS    	CHART            	APP VERSION	DESCRIPTION     
1       	Wed Oct  7 21:32:30 2026	superseded	notes-chart-0.1.0	1.0        	Install complete
2       	Wed Oct  7 21:32:33 2026	deployed  	notes-chart-0.1.0	1.0        	Upgrade complete
```

### Bad upgrade (step 13)

```text
$ helm upgrade notes-dev notes-chart --set image.tag=broken-tag-does-not-exist
Release "notes-dev" has been upgraded. Happy Helming!
NAME: notes-dev
LAST DEPLOYED: Wed Oct  7 21:32:55 2026
NAMESPACE: default
STATUS: deployed
REVISION: 3
DESCRIPTION: Upgrade complete
TEST SUITE: None

$ kubectl get pods
NAME                                READY   STATUS             RESTARTS   AGE
notes-dev-deploy-57c655ccc5-2hvdh   1/1     Running            0          53s
notes-dev-deploy-844c88d6df-ws7kb   0/1     ImagePullBackOff   0          31s

$ kubectl describe pod -l app=notes-dev | grep -E 'Image:|Reason:' | sort | uniq -c
      1     Image:          nginx:1.25
      1     Image:          nginx:broken-tag-does-not-exist
      1       Reason:       ImagePullBackOff

$ helm history notes-dev
REVISION	UPDATED                 	STATUS    	CHART            	APP VERSION	DESCRIPTION     
1       	Wed Oct  7 21:32:30 2026	superseded	notes-chart-0.1.0	1.0        	Install complete
2       	Wed Oct  7 21:32:33 2026	superseded	notes-chart-0.1.0	1.0        	Upgrade complete
3       	Wed Oct  7 21:32:55 2026	deployed  	notes-chart-0.1.0	1.0        	Upgrade complete
```

Helm marks revision 3 `deployed` while the new pod cannot pull its image. Without `--wait`, Helm
reports that the API server accepted the objects. It does not check pod health. One old pod stays
`Running`, so the app still answers. Why only one, and not three, is in [Findings](#findings).

### Rollback to revision 2 (step 14)

```text
$ helm rollback notes-dev 2 --wait --timeout 3m
Rollback was a success! Happy Helming!

$ kubectl get pods -o 'custom-columns=POD:.metadata.name,STATUS:.status.phase,IMAGE:.spec.containers[0].image'
POD                                 STATUS    IMAGE
notes-dev-deploy-57c655ccc5-2hvdh   Running   nginx:1.25
notes-dev-deploy-57c655ccc5-8bd8h   Running   nginx:1.25
notes-dev-deploy-57c655ccc5-ltdxc   Running   nginx:1.25

$ helm history notes-dev
REVISION	UPDATED                 	STATUS    	CHART            	APP VERSION	DESCRIPTION     
1       	Wed Oct  7 21:32:30 2026	superseded	notes-chart-0.1.0	1.0        	Install complete
2       	Wed Oct  7 21:32:33 2026	superseded	notes-chart-0.1.0	1.0        	Upgrade complete
3       	Wed Oct  7 21:32:55 2026	superseded	notes-chart-0.1.0	1.0        	Upgrade complete
4       	Wed Oct  7 21:33:26 2026	deployed  	notes-chart-0.1.0	1.0        	Rollback to 2   

$ kubectl get configmap notes-dev-config -o jsonpath='{.data}'
{"APP_NAME":"notes-app","ENVIRONMENT":"production"}

$ kubectl exec deploy/notes-dev-deploy -- printenv ENVIRONMENT
production
```

The rollback restored all of revision 2: three pods, nginx:1.25, and `ENVIRONMENT=production`.

### Clean up (step 15)

```text
$ helm uninstall notes-dev
release "notes-dev" uninstalled

$ kubectl get pods
No resources found in default namespace.

$ kubectl get services
NAME         TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
kubernetes   ClusterIP   10.96.0.1    <none>        443/TCP   45m
```

---

## Findings

### 1. Step 13 of the mini project also drops the production values

`helm upgrade` without `-f values-prod.yaml` starts again from `values.yaml`. It does not keep the
values of the previous revision. So the bad upgrade in step 13 does two things: it sets the broken
tag, and it sets `replicaCount` back to 1 and `ENVIRONMENT` back to `development`. That is why only
one old pod was left in step 13. The Deployment wanted 1 replica, not 3.

To change only the tag, use one of these:

```bash
helm upgrade notes-dev notes-chart -f notes-chart/values-prod.yaml --set image.tag=broken-tag-does-not-exist
helm upgrade notes-dev notes-chart --reuse-values --set image.tag=broken-tag-does-not-exist
```

### 2. My first mini-project run timed out on an image pull

In the first run, `helm install --wait --timeout 3m` failed. The Docker Hub pull of `nginx:1.24` was
slow, and the pod reached `ImagePullBackOff`:

```text
$ helm install notes-dev notes-chart --wait --timeout 3m
Error: INSTALLATION FAILED: resource Deployment/default/notes-dev-deploy not ready. status: InProgress, message: Available: 0/1
context deadline exceeded

$ kubectl get pods
NAME                                READY   STATUS             RESTARTS   AGE
notes-dev-deploy-578b55fc88-sd7br   0/1     ImagePullBackOff   0          3m
```

The pull later succeeded. A test pod with `nginx:1.24` then printed `Container image "nginx:1.24"
already present on machine`. I uninstalled the release and ran the whole mini project again. The
Task 3 section shows the second run.

### 3. Helm v3 to v4 differences I hit

| Area | Helm v3 | Helm v4.2.2 (what I saw) |
| :--- | :--- | :--- |
| Auto rollback flag | `--atomic` | `--rollback-on-failure`. `--atomic` still works but prints a deprecation warning. |
| `helm status` | Resources only with `--show-resources` | Lists resources and pods by default |
| `--wait` | Polls resources | Uses kstatus. Errors look like `resource Deployment/default/hello not ready. status: InProgress`. |
| `helm create` skeleton | No Gateway API template | Adds `templates/httproute.yaml` |
| `helm list -a` | Shows all releases, including failed | Removed: `Error: unknown shorthand flag: 'a' in -a`. `helm list` shows all states by default. |
| Course README install line | Installs Helm 3 | I skipped it. Every course command worked on v4. |

### 4. A failed release shows up as two rows

With `--rollback-on-failure`, one failed upgrade adds two revisions: the failed upgrade and the
rollback. Revisions 6 and 7 above show this.

### 5. Course folder traps (from the course run sheet)

- `04-chart-yaml` has no `templates/`. It is metadata only.
- `05-values-yaml` has no `Chart.yaml`. Use it with `-f` against the chart in folder 03.
- Folder 05 `values-prod.yaml` asks for `nginx:v2.0.0`. That tag does not exist.
