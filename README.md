# Steps for the Demo
## Install Argo CD
```
kubectl create ns argocd
kubectl create -f https://raw.githubusercontent.com/argoproj/argo-cd/v2.14.8/manifests/install.yaml -n argocd
```
## Install Kyverno
```
kubectl create ns kyverno
kubectl create -f https://github.com/kyverno/kyverno/releases/download/v1.13.0/install.yaml -n kyverno
```

## Enable Apps in Any Namespace feature
Each tenant gets their own namespace where they will be able to create their Argo Application. Have a fixed pattern that all tenant namespace would follow. eg: have all namespaces have suffix `-tenant-ns`, so that you can have the glob pattern `*-tenant-ns` specified in Argo CD configuration.
```
kubectl patch cm argocd-cmd-params-cm -n argocd -p '{"data":{"application.namespaces": "*-tenant-ns"}}'
```
```
kubectl apply -k "https://github.com/argoproj/argo-cd/examples/k8s-rbac/argocd-server-applications/?ref=v2.14.8"
```
## Enable the application sync with impersonation feature
```
kubectl patch cm argocd-cm -n argocd -p '{"data":{"application.sync.impersonation.enabled": "true"}}'
```
## Disable the application sync with impersonation feature
```
kubectl patch cm argocd-cm -n argocd -p '{"data":{"application.sync.impersonation.enabled": "false"}}'
```

# Tenant Onboarding process
## Approach 1: Policy enforced AppProjects
### Create the namespace for the tenant being onboarded
```
kubectl create ns anandf-tenant-ns
kubectl create ns jfischer-tenant-ns
```
Note: Ensure that the namespace matches the pattern used when enabling apps in any namespace feature
### Create and Configure AppProjects for each tenant
#### Get the Argo CD server admin password
```
PASSWORD=$(kubectl get secret argocd-initial-admin-secret -o jsonpath='{.data.password}' | base64 -d)
```
#### Login to argocd instance via CLI
```
argocd login <instance_host> --username admin --password ${PASSWORD}
```
#### Create the AppProject one per tenant
```
argocd proj create anandf-tenant
argocd proj create jfischer-tenant
```
#### Configure the AppProject to allow the destination namespaces
Allow only destination namespace with tenant prefix matching their ID.
```
argocd proj add-destination anandf-tenant https://kubernetes.default.svc 'anandf-*'
argocd proj add-destination jfischer-tenant https://kubernetes.default.svc 'jfischer-*'
```
#### Configure the AppProject to allow argo applications only from their respective namespaces.
Allow only applications defined in their namespace to be used for each project
```
argocd proj add-source-namespace anandf-tenant anandf-tenant-ns
argocd proj add-source-namespace jfischer-tenant jfischer-tenant-ns
```
#### Configure the AppProject to allow any source repository
```
argocd proj add-source anandf-tenant 'https://github.com/anandf/*.git'
argocd proj add-source jfischer-tenant 'https://github.com/jfischer/*.git'
```
#### Configure the AppProject to allow creation of Namespace resource
For tenant admins to be able to create namespaces as part of their application sync, allow users to create namespace.
```
argocd proj allow-cluster-resource anandf-tenant "" "Namespace"
argocd proj allow-cluster-resource jfischer-tenant "" "Namespace"
```

#### Add Destination Service accounts to each tenant
```
argocd proj add-destination-service-account anandf-tenant https://kubernetes.default.svc 'anandf-*' 'anandf-admin-sa'
argocd proj add-destination-service-account jfischer-tenant https://kubernetes.default.svc 'jfischer-*' 'jfischer-admin-sa'
```
### Create roles for each tenant in AppProject
Create roles in the AppProject - read-only, admin, user, pipeline
```
spec:
  roles:
  - description: Read Only
    name: read-only
    policies:
    - p, proj:anandf-tenant:read-only, applications, get, anandf-tenant-ns/*, allow
  - description: tenant admins
    name: admin
    policies:
    - p, proj:anandf-tenant:admin, applications, *, anandf-tenant-ns/*, allow
  - description: tenant users
    name: user
    policies:
    - p, proj:anandf-tenant:user, applications, get, anandf-tenant-ns/*, allow
    - p, proj:anandf-tenant:user, applications, sync, anandf-tenant-ns/*, allow
  - description: pipeline accounts
    name: pipeline
    policies:
    - p, proj:anandf-tenant:pipeline, applications, get, anandf-tenant-ns/*, allow
    - p, proj:anandf-tenant:pipeline, applications, sync, anandf-tenant-ns/*, allow
```

### Create kyverno policy to disable applications with default project
```
kubectl create -f kyverno/disable-default-appproject.yaml
```
Try creating an application as below and ensure that the creation fails with error message from kyverno admission controller.
```
kubectl apply -n anandf-tenant-ns -f - <<EOF
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
    name: guestbook
spec:
  destination:
    namespace: anandf-guestbook
    server: https://kubernetes.default.svc
  project: default
  source:
    directory:
      recurse: true
    path: guestbook
    repoURL: https://github.com/argoproj/argocd-example-apps.git
  syncPolicy:
    automated: {}
    syncOptions:
    - CreateNamespace=true
EOF
```

### Create kyverno policy to ensure that applications refer only to projects in their tenant.
```
kubectl create -f kyverno/restrict-application-project-reference.yaml
```
Try creating an application as below and ensure that the creation fails with error message from kyverno admission controller.
```
kubectl apply -n anandf-tenant-ns -f - <<EOF
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
    name: guestbook
spec:
  destination:
    namespace: jfischer-guestbook
    server: https://kubernetes.default.svc
  project: anandf-tenant
  source:
    directory:
      recurse: true
    path: guestbook
    repoURL: https://github.com/argoproj/argocd-example-apps.git
  syncPolicy:
    automated: {}
    syncOptions:
    - CreateNamespace=true
EOF
```
##### Expected Error message
```
Error from server: error when applying patch:
{"metadata":{"annotations":{"kubectl.kubernetes.io/last-applied-configuration":"{\"apiVersion\":\"argoproj.io/v1alpha1\",\"kind\":\"Application\",\"metadata\":{\"annotations\":{},\"name\":\"guestbook\",\"namespace\":\"anandf-tenant-ns\"},\"spec\":{\"destination\":{\"namespace\":\"anandf-guestbook\",\"server\":\"https://kubernetes.default.svc\"},\"project\":\"jfischer-tenant\",\"source\":{\"directory\":{\"recurse\":true},\"path\":\"guestbook\",\"repoURL\":\"https://github.com/argoproj/argocd-example-apps.git\"},\"syncPolicy\":{\"automated\":{},\"syncOptions\":[\"CreateNamespace=true\"]}}}\n"}},"spec":{"project":"jfischer-tenant"}}
to:
Resource: "argoproj.io/v1alpha1, Resource=applications", GroupVersionKind: "argoproj.io/v1alpha1, Kind=Application"
Name: "guestbook", Namespace: "anandf-tenant-ns"
for: "STDIN": error when patching "STDIN": admission webhook "validate.kyverno.svc-fail" denied the request:

resource Application/anandf-tenant-ns/guestbook was blocked due to the following policies

restrict-application-project-reference:
  check-application-refers-allowed-project: Application must refer only projects allowed
    for the tenant.
```

### Create kyverno policy to allow creation of application with destination namespace having tenant ID as the prefix
```
kubectl create -f kyverno/restrict-destination-namespace.yaml
```
```
kubectl apply -n anandf-tenant-ns -f - <<EOF
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
    name: guestbook
spec:
  destination:
    namespace: anandf-guestbook
    server: https://kubernetes.default.svc
  project: anandf-tenant
  source:
    directory:
      recurse: true
    path: guestbook
    repoURL: https://github.com/argoproj/argocd-example-apps.git
  syncPolicy:
    automated: {}
    syncOptions:
    - CreateNamespace=true
EOF
```
##### Expected error message
```
Error from server: error when applying patch:
{"metadata":{"annotations":{"kubectl.kubernetes.io/last-applied-configuration":"{\"apiVersion\":\"argoproj.io/v1alpha1\",\"kind\":\"Application\",\"metadata\":{\"annotations\":{},\"name\":\"guestbook\",\"namespace\":\"anandf-tenant-ns\"},\"spec\":{\"destination\":{\"namespace\":\"jfischer-guestbook\",\"server\":\"https://kubernetes.default.svc\"},\"project\":\"anandf-tenant\",\"source\":{\"directory\":{\"recurse\":true},\"path\":\"guestbook\",\"repoURL\":\"https://github.com/argoproj/argocd-example-apps.git\"},\"syncPolicy\":{\"automated\":{},\"syncOptions\":[\"CreateNamespace=true\"]}}}\n"}},"spec":{"destination":{"namespace":"jfischer-guestbook"}}}
to:
Resource: "argoproj.io/v1alpha1, Resource=applications", GroupVersionKind: "argoproj.io/v1alpha1, Kind=Application"
Name: "guestbook", Namespace: "anandf-tenant-ns"
for: "STDIN": error when patching "STDIN": admission webhook "validate.kyverno.svc-fail" denied the request:

resource Application/anandf-tenant-ns/guestbook was blocked due to the following policies

restrict-destination-namespace:
  check-destination-namespace-has-tenant-id-prefix: Destination namespace of Application
    must match the tenant id.
```
### Create an application in each tenant
```
argocd app create anandf-tenant-ns/guestbook \
  --project anandf-tenant \
  --repo https://github.com/argoproj/argocd-example-apps.git \
  --path guestbook \
  --dest-namespace anandf-guestbook \
  --dest-server https://kubernetes.default.svc \
  --directory-recurse \
  --sync-policy auto \
  --sync-option CreateNamespace=true
```
```
argocd app create jfischer-tenant-ns/guestbook \
  --project jfischer-tenant \
  --repo https://github.com/argoproj/argocd-example-apps.git \
  --path guestbook \
  --dest-namespace jfischer-guestbook \
  --dest-server https://kubernetes.default.svc \
  --directory-recurse \
  --sync-policy auto \
  --sync-option CreateNamespace=true
```
## Approach 2: Application sync using impersonation

## Enable the application sync with impersonation feature
```
kubectl patch cm argocd-cm -n argocd -p '{"data":{"application.sync.impersonation.enabled": "true"}}'
```
### Create and Configure AppProjects for each tenant
#### Get the Argo CD server admin password
```
PASSWORD=$(kubectl get secret argocd-initial-admin-secret -o jsonpath='{.data.password}' | base64 -d)
```
#### Login to argocd instance via CLI
```
argocd login <instance_host> --username admin --password ${PASSWORD}
```
#### Create the AppProject one per tenant
```
argocd proj create anandf-tenant
argocd proj create jfischer-tenant
```
#### Configure the AppProject to allow the destination namespaces
Allow only destination namespace with tenant prefix matching their ID.
```
argocd proj add-destination anandf-tenant https://kubernetes.default.svc 'anandf-*'
argocd proj add-destination jfischer-tenant https://kubernetes.default.svc 'jfischer-*'
```
#### Configure the AppProject to allow argo applications only from their respective namespaces.
Allow only applications defined in their namespace to be used for each project
```
argocd proj add-source-namespace anandf-tenant anandf-tenant-ns
argocd proj add-source-namespace jfischer-tenant jfischer-tenant-ns
```
#### Configure the AppProject to allow any source repository
```
argocd proj add-source anandf-tenant 'https://github.com/anandf/*.git'
argocd proj add-source jfischer-tenant 'https://github.com/jfischer/*.git'
```
#### Configure the AppProject to allow creation of Namespace resource
For tenant admins to be able to create namespaces as part of their application sync, allow users to create namespace.
```
argocd proj allow-cluster-resource anandf-tenant "" "Namespace"
argocd proj allow-cluster-resource jfischer-tenant "" "Namespace"
```

#### Add Destination Service accounts to each tenant
```
argocd proj add-destination-service-account anandf-tenant https://kubernetes.default.svc 'anandf-*' 'anandf-tenant-ns:anandf-admin-sa'
argocd proj add-destination-service-account jfischer-tenant https://kubernetes.default.svc 'jfischer-*' 'jfischer-tenant-ns:jfischer-admin-sa'
```

### Create the service account for impersonation
```
kubectl create sa -n anandf-tenant-ns anandf-admin-sa
kubectl create sa -n jfischer-tenant-ns jfischer-admin-sa
```

### Create the role and rolebinding for the service account
```
kubectl create role -n anandf-guestbook anandf-admin --verb='*' --resource="deployment.apps" --resource="service"
kubectl create rolebinding -n anandf-guestbook anandf-admin-sa-rb --role anandf-admin --serviceaccount anandf-tenant-ns:anandf-admin-sa
```
```
kubectl create role -n jfischer-guestbook jfischer-admin --verb='*' --resource="deployment.apps" --resource="service"
kubectl create rolebinding -n jfischer-guestbook jfischer-admin-sa-rb --role jfischer-admin --serviceaccount jfischer-tenant-ns:jfischer-admin-sa
```
### Create an application in each tenant
```
argocd app create anandf-tenant-ns/guestbook \
  --project anandf-tenant \
  --repo https://github.com/argoproj/argocd-example-apps.git \
  --path guestbook \
  --dest-namespace anandf-guestbook \
  --dest-server https://kubernetes.default.svc \
  --directory-recurse \
  --sync-policy auto
```
```
argocd app create jfischer-tenant-ns/guestbook \
  --project jfischer-tenant \
  --repo https://github.com/argoproj/argocd-example-apps.git \
  --path guestbook \
  --dest-namespace jfischer-guestbook \
  --dest-server https://kubernetes.default.svc \
  --directory-recurse \
  --sync-policy auto
```