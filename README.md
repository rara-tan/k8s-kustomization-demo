# What is This Repo
To demonstrate how to use Kubernetes kusomization.

# Pre Requirements
## install kubectl & kind
https://kubernetes.io/docs/tasks/tools/

## Create Kubernetes Cluster
I recommend a kind cluster.(any k8s cluster is OK)

```bash
$ kind create cluster -n kustomization-demo
```

# Basic Usage
Basic usage of kustomization is deploying several Kubernetes resources at once.  
At a test, Let's apply deployment and service files at onece.

```bash
$ kubectl apply -k ./basic
```

After applying, you can find a deployment(test-app) and a service(test-app) on your kubernetes cluster.
```bash
$ kubectl get deploy
NAME       READY   UP-TO-DATE   AVAILABLE   AGE
test-app   1/1     1            1           39s

$ kubectl get svc
NAME         TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)   AGE
kubernetes   ClusterIP   10.96.0.1      <none>        443/TCP   55m
test-svc     ClusterIP   10.96.83.182   <none>        80/TCP    39s
```

If you want to check yaml file that will be created by kustomization, you run a dry-run command.

```bash
$ kubectl apply -k ./basic --dry-run=client -o yaml
```

Or if you installed kustomize command, you can use kustomize build commnad.

```bash
$ kustomize build ./basic
```

# patch replicas
You can change the number of deployment replica by using kustomization.

`patch/kustomization.yaml`
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
- ./service.yaml
- ./deployment.yaml
replicas:
- name: patch-replica
  count: 3
```

By adding replicas to the kusomization file, you can change the number of replica count.

```bash
$ kubectl apply -k ./patch-replica
service/patch-replica-svc created
deployment.apps/patch-replica created

$ kubectl get deploy
NAME            READY   UP-TO-DATE   AVAILABLE   AGE
patch-replica   3/3     3            3           44s
test-app        1/1     1            1           9m38s
```

# patch other parametes
To change other parameters, Kustomize supports `patch` to modify Kubernetes reosources.

For example, you can change a namespace and add a label by following codes.

`patch-other-params/kustomization.yaml`
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
- ./deployment.yaml
- ./namespace.yaml
patches:
  - target:
      version: v1
      group: apps
      kind: Deployment
      name: patch-other
    patch: |-
      - op: replace
        path: /metadata/namespace
        value: patch-other-parameters
      - op: add
        path: "/metadata/labels/patch"
        value: patched-value
```

```bash
$ kubectl apply -k ./patch-other-params
namespace/patch-other-parameters created
service/patch-other-svc created
deployment.apps/patch-other created

$ kubectl get deploy -n patch-other-parameters
NAME          READY   UP-TO-DATE   AVAILABLE   AGE
patch-other   1/1     1            1           23s
```

# patch by files
Instead of embedding a json patch expression, you can use a yaml file using the path field.

`patch-by-files/kustomization.yaml`
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
- ./deployment.yaml
- ./namespace.yaml
patches:
  - target:
      version: v1
      group: apps
      kind: Deployment
      name: patch-by-file
    path: ./patch.yaml
```

```bash
$ kubectl apply -k ./patch-by-files
namespace/patch-by-files created
deployment.apps/patch-by-file created

$ kubectl get deploy -n patch-by-files
NAME            READY   UP-TO-DATE   AVAILABLE   AGE
patch-by-file   1/1     1            1           87s
```

# Strategic Merge Patch
You can use an incomplete YAML file to patch.

For example, you can add resource requests and limits to the deployment by following expressions.

`strategic-merge-patch/patch.yaml`
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: strategic-merge-patch
spec:
  template:
    spec:
      containers:
      - name: nginx
        resources:
          limits:
            cpu: "300m"
            memory: "512Mi"
          requests:
            cpu: "300m"
            memory: "512Mi"
```

`strategic-merge-patch/kustomization.yaml`
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
- ./deployment.yaml
patchesStrategicMerge:
  - patch.yaml
```

```bash
$ kubectl apply -k ./strategic-merge-patch 
deployment.apps/strategic-merge-patch created

$ kubectl get deploy strategic-merge-patch -o yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: strategic-merge-patch
  ...
spec:
  ...
    spec:
      containers:
      - image: nginx
        imagePullPolicy: Always
        name: nginx
        resources:
          limits:
            cpu: 300m
            memory: 512Mi
          requests:
            cpu: 300m
            memory: 512Mi
...
```