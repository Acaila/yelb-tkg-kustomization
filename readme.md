# How to deploy YELB demo app on to TKG

Assumes that images are already pushed into local harbor and supervisor namespace is setup/ready

## In GUI
1.   Create namespace
   1. Add AD group as Owner
   1. Add Storage policy
   1. Add content library

## Login to supervisor

```bash
kubectl vsphere login --insecure-skip-tls-verify --server <IP>
```

## Create Harbor Cert Secret

We need to create a secret of the on prem harbor cert, we do this by base64 encoding the cert (twice) and create the secret

```powershell
$NS       = "vcf-w1-tkg1-demo"
$CLUSTER  = "demo-w1c1"
$cert     = "harbor-ca.crt"

$once =  [Convert]::ToBase64String([IO.File]::ReadAllBytes($Cert))
$twice = [Convert]::ToBase64String([Text.Encoding]::ASCII.GetBytes($once))

@"
apiVersion: v1
kind: Secret
metadata:
  name: $CLUSTER-user-trusted-ca-secret
  namespace: $NS
type: Opaque
data:
  harbor-ca: $twice
"@ | kubectl apply -f -
```

## Deploy TKG cluster
Create a demo-w1c1.yaml file with the following contents.  This YAML states we are deploying a `TanzuKubernetesCluster` with 1 controller and 2 workers that are all t-shirt size of `best-effort-small` and using the 1.30.1 tkr image (from the content library)

```yaml
apiVersion: cluster.x-k8s.io/v1beta1
kind: Cluster
metadata:
  name: demo-w1c1
  namespace: vcf-w1-tkg1-demo
spec:
  clusterNetwork:
      services:
          cidrBlocks: ["10.96.0.0/12"]
      pods:
          cidrBlocks: ["100.96.0.0/11"]
      serviceDomain: "cluster.local"
  topology:
      class: tanzukubernetescluster
      version: v1.30.1---vmware.1-fips-tkg.5
      controlPlane:
          replicas: 1
      workers:
          machineDeployments:
              - class: node-pool
                name: node-pool-01
                replicas: 2
      variables:
          - name: vmClass
            value: best-effort-small
          - name: storageClass 
            value: vcf-w1-tkg1-default
          - name: defaultStorageClass
            value: vcf-w1-tkg1-default
          - name: trust
            value:
              additionalTrustedCAs:
              - name: harbor-ca
```

Save and run apply to deploy:

```bash
kubectl apply -f .\demo-w1c1.yaml
```

Watch deployment in both the GUI and via `kubectl get clusters` command:

```bash
kubectl -n vcf-w1-tkg1-demo get clusters.cluster.x-k8s.io -o wide
```

Output:

```powershell
PS:> kubectl -n vcf-w1-tkg1-demo get clusters.cluster.x-k8s.io -o wide
NAME        CLUSTERCLASS             PHASE         AGE     VERSION
demo-w1c1   builtin-generic-v3.1.0   Provisioned   6m19s   v1.30.1+vmware.1-fips
```

## Login to the new TKG cluster

Now we need to login to the TKG cluster, using a similar command to when we logged into the supervisor:

```powershell
kubectl vsphere login --insecure-skip-tls-verify --server <IP> `
   --tanzu-kubernetes-cluster-name demo-w1c1 `
   --tanzu-kubernetes-cluster-namespace vcf-w1-tkg1-demo


Username: <USERNAME>
Password: 
Logged in successfully.

You have access to the following contexts:
   <SUPER IP>
   demo-w1c1
   vcf-w1-tkg1-demo

If the context you wish to use is not in this list, you may need to try
logging in again later, or contact your cluster administrator.

To change context, use `kubectl config use-context <workload name>`
```
### Switch context and create demo namespace (Inside TKG)

Now we create a namespace inside the TKG cluster for our demo app to be deployed into

```powershell
k config use-context demo-w1c1
k create namespace yelb-demo
```

### Create Harbor secret
Currently Harbor requires authentication, so create a secret with our user/pass we can reference in our deployment files:

```powershell
kubectl -n yelb-demo create secret docker-registry regcred `
  --docker-server harbor.lab.local `
  --docker-username='USERNAME' `
  --docker-password='PASSWORD' `
  --docker-email='harbor@lab.local'
```
**Note**: This secret only allows you to pull down the yelb images and having it here in plain text is just for convienience.  We could change the images to be public on harbor and not require a password instead.

We can make this secret the default for this namespace so its used for all image pulls in this namespace.

```powershell
kubectl -n yelb-demo patch serviceaccount default -p '{"imagePullSecrets":[{"name":"regcred"}]}'
```
# Build patch files to modify deployment to work on TKG

Now we can deploy the yelb app using our local harbor for images. However we have to make quite a few changes to the kubernetes example YAML provided on the yelb github page.

Our TKG cluster enforces pod security so we must explicetly state in our deployment YAML that we are not running the pods as root and which parts of the containers filesystem are writeable.

The main change is that we override the default nginx.conf to use port 8080 and set the html location to the yelb app under /clarity-seed/prod/dist

We also mount and set a few directory permissions for the db (postgres) and redis apps so they can run as non root users.

We can apply these changes as a Kubernetes Kustomization overlay.  Essentailly we tell TKG to apply the yelb YAML file from github, but with some targetted changes to allow it to run air gapped and with enhanced security that TKG requires.

To do this we need a copy of the yelb kubernetes deployment file from github:

https://github.com/mreferre/yelb/blob/master/deployments/platformdeployment/Kubernetes/yaml/yelb-k8s-loadbalancer.yaml

## Create directory structure
Create the following folder structure and place this file inside the `upstream` folder:

```powershell
mkdir "tkg-deploy\overlay-tkg\upstream" -Force
```

## Create kustomization.yaml

Next create a kuzomization.yaml file and save it in the `overlay-tkg` folder.

This file outlies the changes that we will make to the reference yelb-k8s-loadbalancer.yaml file, using it as a base.

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
# Reference local copy of https://github.com/mreferre/yelb/blob/master/deployments/platformdeployment/Kubernetes/yaml/yelb-k8s-loadbalancer.yaml
resources:
  - ./upstream/yelb-k8s-loadbalancer.yaml
namespace: yelb-demo
images:
  - name: mreferre/yelb-ui
    newName: harbor.lab.local/vks-yelb/yelb
    newTag: yelb-ui-0.10
  - name: mreferre/yelb-db
    newName: harbor.lab.local/vks-yelb/yelb
    newTag: yelb-db-0.6
  - name: mreferre/yelb-appserver
    newName: harbor.lab.local/vks-yelb/yelb
    newTag: yelb-appserver-0.7
  - name: redis
    newName: harbor.lab.local/vks-yelb/yelb
    newTag: redis-4.0.2

configMapGenerator:
  - name: yelb-ui-nginx
    behavior: create
    files:
      - default.conf=./yelb-ui-nginx.conf

patches:
  - path: ./patch-yelb-ui-deploy.yaml
    target:
      kind: Deployment
      name: yelb-ui
  - path: ./patch-yelb-ui-svc.yaml
    target:
      kind: Service
      name: yelb-ui
  - path: ./patch-yelb-db-deploy.yaml
    target:
      kind: Deployment
      name: yelb-db
  - path: ./patch-yelb-appserver-deploy.yaml
    target:
      kind: Deployment
      name: yelb-appserver
  - path: ./patch-yelb-redis-deploy.yaml
    target:
      kind: Deployment
      name: redis-server
  - target:
      kind: Deployment
      labelSelector: app
    patch: |-
      apiVersion: apps/v1
      kind: Deployment
      metadata:
        name: dummy
      spec:
        template:
          spec:
            imagePullSecrets:
              - name: regcred
```

## Yelb-UI changes

The yelb-ui component is a container running nginx with the yelb static images and clarity UI images pre-installed.

### nginx.conf

Create a new nginx.conf file that changes the port to 8080 and the root directory to the yelb app under `/clarity-seed/prod/dist`.  Save this file as `yelb-ui-nginx.conf` in the same `overlay-tkg` folder

```conf
server {
    listen 8080;
    server_name _;

    root /clarity-seed/prod/dist;
    index index.html;

    location / {
        try_files $uri $uri/ /index.html;
    }

    location /api {
        proxy_pass          http://yelb-appserver:4567;
        proxy_http_version  1.1;
        proxy_set_header    Host $host;
        proxy_set_header    X-Real-IP $remote_addr;
        proxy_set_header    X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header    X-Forwarded-Proto $scheme;
      }
}
```
### Service override
Because we're changing the Nginx port, we need to update the exposed service entry point to target this port.  We do this in the `patch-yelb-ui-svc.yaml` file with the following contents:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: yelb-ui
spec:
  type: LoadBalancer
  ports:
    - port: 80
      protocol: TCP
      targetPort: 8080 # Maps external 80 to new interal 8080 for nginx pod
```

### Deployment override

Finally we update the deployment component to include the required pod security properties, and mount writeable directories where needed for these containers to run with enhanced security and non root privelges.

File: `patch-yelb-ui-deploy.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: yelb-ui
spec:
  template:
    spec:
      # Pod-level security context required values
      securityContext:
        runAsNonRoot: true
        runAsUser: 1001
        runAsGroup: 1001
        seccompProfile:
          type: RuntimeDefault
      containers:
      - name: yelb-ui
        # Use 8080, service will map 80 -> 8080
        ports:
        - containerPort: 8080
        # Override entrypoint to skip the image's sed-in/etc script and allow non root exec
        command: ["/bin/sh", "-c"]
        args:
          - |
            mkdir -p /var/cache/nginx/client_temp /var/run && \
            nginx -g 'daemon off;'
        securityContext:
          allowPrivilegeEscalation: false
          readOnlyRootFilesystem: false # nginx needs writable cache/temp
          runAsNonRoot: true
          runAsUser: 1001
          runAsGroup: 1001
          capabilities:
            drop: ["ALL"]
          seccompProfile:
            type: RuntimeDefault
        volumeMounts:
          - name: nginx-cache
            mountPath: /var/cache/nginx
          - name: nginx-run
            mountPath: /var/run
          - name: nginx-conf
            mountPath: /etc/nginx/conf.d/default.conf
            subPath: default.conf
      volumes:
        - name: nginx-cache
          emptyDir: {}
        - name: nginx-run
          emptyDir: {}
        - name: nginx-conf
          configMap:
            name: yelb-ui-nginx
```

## Other deployment overrides

We apply similar overrides to the other deployment components

### File: `patch-yelb-redis-deploy.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: redis-server
spec:
  template:
    spec:
      securityContext:
        runAsNonRoot: true
        runAsUser: 1001
        runAsGroup: 1001
        fsGroup: 1001
        seccompProfile:
          type: RuntimeDefault
      containers:
      - name: redis-server
        command: ["redis-server"]
        args: ["--save", "", "--appendonly", "no"]
        securityContext:
          allowPrivilegeEscalation: false
          readOnlyRootFilesystem: true
          capabilities:
            drop: ["ALL"]
        volumeMounts:
        - name: data
          mountPath: /data
      volumes:
      - name: data
        emptyDir: {}
```

### File: patch-yelb-appserver-deploy.yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: yelb-appserver
spec:
  template:
    spec:
      # Pod Level Security requirements
      securityContext:
        runAsNonRoot: true
        runAsUser: 1001
        runAsGroup: 1001
        seccompProfile:
          type: RuntimeDefault
      containers:
        - name: yelb-appserver
          securityContext:
            allowPrivilegeEscalation: false
            readOnlyRootFilesystem: true
            runAsNonRoot: true
            runAsUser: 1001
            runAsGroup: 1001
            capabilities:
              drop: ["ALL"]
            seccompProfile:
              type: RuntimeDefault
```

### File: patch-yelb-db-deploy.yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: yelb-db
spec:
  template:
    spec:
      # Pod Level Security requirements
      securityContext:
        runAsNonRoot: true
        runAsUser: 999
        runAsGroup: 999
        fsGroup: 999
        fsGroupChangePolicy: OnRootMismatch
        seccompProfile:
          type: RuntimeDefault
      containers:
        - name: yelb-db
          env:
            - name: PGDATA
              value: /var/lib/postgresql/data/pgdata
          securityContext:
            allowPrivilegeEscalation: false
            readOnlyRootFilesystem: false # postgres needs to write to /var/lib/postgres
            runAsNonRoot: true
            runAsUser: 999
            runAsGroup: 999
            capabilities:
              drop: ["ALL"]
            seccompProfile:
              type: RuntimeDefault
          volumeMounts:
            - name: pgdata
              mountPath: /var/lib/postgresql/data
            - name: pgrun
              mountPath: /var/run/postgresql
      volumes:
        - name: pgdata
          emptyDir: {}
        - name: pgrun
          emptyDir: {}
```
## Deploy using customisations.

Apply the kustomization with the following command and then watch nodes come up:

```powershell
k apply -k .\overlay-tkg\
```

Watch the deployment with the following command:

```powershell
k -n yelb-demo get deploy,po -o wide
```

When it completes call get service (svc) to see what external IP was assigned to the frontend load balancer:

```powershell
k -n yelb-demo get svc yelb-ui
```

You will see output similar too:

```powershell
PS V:\Tanzu> k -n yelb-demo get svc yelb-ui
NAME      TYPE           CLUSTER-IP       EXTERNAL-IP       PORT(S)        AGE
yelb-ui   LoadBalancer   10.100.172.152   <EEXTERNAL IP>    80:32300/TCP   30s
```

We can load the external IP in a browser to see the application running.
