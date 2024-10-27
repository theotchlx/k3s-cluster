# k3s-cluster
My personal k3s cluster deployment with my applications

## Prerequisites
You have kubectl on your local machine
You have a configured Debian Linux server.
`open-ssh` server, `curl`, `gpg`.

## Manual set up
No automatic infrastructure deployment or set up.
From this point on, you should have knowledge on how to properly set-up the installed programs to have correct security configurations for Docker Engine, K3s and the rest of the apps.
*This setup may not currently reflect such knowledge.*
## Docker Engine installation
On your server machine, as root:
Execute the following.
```bash
# Add Docker's official GPG key:
apt-get update
apt-get install ca-certificates curl
install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/debian/gpg -o /etc/apt/keyrings/docker.asc
chmod a+r /etc/apt/keyrings/docker.asc

# Add the repository to Apt sources:
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/debian \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  tee /etc/apt/sources.list.d/docker.list > /dev/null
apt-get update
```
```bash
apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

### K3s installation
On your server machine, as root:
Execute the following.
```bash
curl -sfL https://get.k3s.io | sh -
```
This installs and sets up K3S ***WITH THE DEFAULTS.***
*OUVRIR LES BONS PORTS SUR LA BOX / OU PLUTÔT TROUVER UNE SOLUTION GENRE UN serveur de redirection AU MILIEU OU JSP... Pour ne pas exposer directement les ports demandés par K3s (sur leur doc)*

Get the kubeconfig at `/etc/rancher/k3s/k3s.yaml`. Copy it on your local machine at `.kube/config` and put the right server IP

### Helm installation
On your local machine, as root:
Execute the following.
```bash
curl https://baltocdn.com/helm/signing.asc | gpg --dearmor | tee /usr/share/keyrings/helm.gpg > /dev/null
apt-get install apt-transport-https --yes
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/helm.gpg] https://baltocdn.com/helm/stable/debian/ all main" | tee /etc/apt/sources.list.d/helm-stable-debian.list
apt-get update
apt-get install helm
```

### ArgoCD installation using Helm
From this point on, you should have knowledge on how to use K3s and Helm to install and configure K3s, Helm, ArgoCD and the rest of the apps properly.
*This setup may not currently reflect such knowledge.*
**PERSONAL NOTE**: *In this section, I somewhat follow https://www.arthurkoziel.com/setting-up-argocd-with-helm/*. See there for more info on some configurations.

On your local machine.

We will make a git repository with our own custom argocd helm chart that wraps around a community-maintained argocd helm chart. Our chart overrides the default values.

Place yourself in your git repository's directory, and execute the following.
```bash
mkdir -p charts/argo-cd
```

Stay in your git repository's directory.
Write the following to `charts/argo-cd/Chart.yaml`:
```yaml
apiVersion: v2
name: argo-cd
version: 1.0.0
dependencies:
  - name: argo-cd
    version: 5.46.8
    repository: https://argoproj.github.io/argo-helm
```

Write the following to `charts/argo-cd/values.yaml`:
```yaml
argo-cd:
  dex:
    enabled: false
  notifications:
    enabled: false
  applicationSet:
    enabled: false
  server:
    extraArgs:
      - --insecure
  controller:
    metrics:
      enabled: true
      serviceMonitor:
        enabled: true
  repoServer:
    resources:
      requests:controller:
    metrics:
      enabled: true
      serviceMonitor:
        enabled: true
```

Let's generate a helm chart lock:
```bash
helm repo add argo-cd https://argoproj.github.io/argo-helm
helm dep update charts/argo-cd/
```

This creates the `charts/argo-cd/Chart.lock` file and the `charts/argo-cd/charts/argo-cd-<version>.tgz` directory and file. The .tgz is only required for the initial installation from our local machine: to avoid accidentally committing it, we can add it to the gitignore file:
```bash
echo "charts/**/charts" >> .gitignore
```

You should commit them if you wish for the dependencies to not be reconstructed from newer upstream versions later, when you or others will execute `helm dep update` again.
If you don't need strict control of dependencies versions (i.e. not a production environment), then committing the .tgz files is bloat; but you will need to update the helm dependencies again.

Personally, I committed them (see line 2):
```bash
git add .
git add -f charts/argo-cd/charts/.
git commit -m "ops(helm charts): add ArgoCD chart"
git push
```

We have pushed our custom chart to our public Git repository.
Now, we can install our helm chart.

Still on your local machine:
We will do the initial installation manually, and then set up ArgoCD to manage itself (by detecting any changes to the helm chart, and synchronizing to it).

Manual initial installation:
For proper/better isolation and management, we will deploy ArgoCD in its own namespace:

```bash
kubectl create namespace argocd
helm install argo-cd charts/argo-cd/ -n argocd
```

Our helm chart doesn't install an Ingress by default. To access the Web UI, we have to port-forward to the argocd-server service on port 443. We expose every IPv4 interfaces (0.0.0.0) in the VM:
```bash
kubectl -n argocd port-forward --address 0.0.0.0 svc/argo-cd-argocd-server 8080:443 &
```
*That command didn't work for me.. Even after restarting the k3s service. I had to execute the command from the VM as root... It may be an issue with the kubeconfig privileges?*

You can now connect to `http://<vm-ip>:8080` to access the Web UI. The default username is admin. The password is auto-generated, we can get it with:
```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" |base64 -d
```

### Deploying applications through ArgoCD
We want to manage our applications declaratively (infrastructure as code), not through the UI or CLI. That means we need to write YAML application manifests, and commit them to our Git repository.
These yaml application manifest would specify the URL to the helm chart, and override values to customize it.

The manifest won't be applied manually via the kubectl CLI. It will be applied automatically by ArgoCD.
We will implement the [app of apps](https://argo-cd.readthedocs.io/en/stable/operator-manual/declarative-setup/#app-of-apps) (to read, as well as [ApplicationSets](https://argo-cd.readthedocs.io/en/stable/user-guide/application-set/)) pattern. We will make and add manually a "root-app" helm chart that renders application manifests. After that, we can just commit application manifests, and they will be deployed automatically.
After adding more applications, there will be lots of duplicated code (like the destination cluster). We can reduce it by putting the values in the charts values file. Another interesting solution for this are ApplicationSets, which we won't see here.

ArgoCD will be in the `argocd` namespace, and our apps will also live  in the `argocd` namespace. Please understand it's not the best security practice to have all your apps in the same namespace, with every default configurations.

On your local machine, in your git repository directory:
```bash
mkdir -p charts/root-app/templates
touch charts/root-app/values.yaml
```

Write the following to `charts/root-app/Chart.yaml`:
```yaml
apiVersion: v2
name: root-app
version: 1.0.0
```

Then, we create the application manifest for our root-app. Make sure to replace repoURL with your public repo's.
Write the following to `charts/root-app/templates/root-app.yaml`:
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: root-app
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/theotchlx/k3s-cluster.git
    path: charts/root-app/
    targetRevision: HEAD
  destination:
    server: https://kubernetes.default.svc
    namespace: argocd
  syncPolicy:
    automated:
      selfHeal: true
```
The above application watches our apps helm chart (under `charts/root-app/`) and if changes are detected, they get synchronized (meaning rendering the chart and applying the resulting manifests on the cluster).

ArgoCD will not use `helm install` to install charts. I will render the chart with `helm template` and then apply the output with `kubectl`. This means we can't run `helm list` on a local machine to get all installed releases.

Let's push our changes:
```bash
git add charts/root-app/
git commit -m "ops(app of apps): add root-app"
git push
```

And apply the manifest in our K3s cluster. We have to do it manually this first time. Later, ArgoCD will manage and synchronize the root-app automatically.
```bash
helm template charts/root-app/ |kubectl apply -n argocd -f -
```
ArgoCD expects its application resources to be applied in its namespace.
ArgoCD will deploy the resources that the "root-app" application manages in the argocd namespace also, as specified in the template.

Now, ArgoCD will manage our apps.

### Installing other applications
We will now take a real example and deploy an Apache Airflow Helm chart on our cluster using ArgoCD.

First, we will make our own custom Airflow Helm chart, with our own values (with a HA postgresql, with a chart from bitnami).
```bash
mkdir charts/airflow
```

Write the following to `charts/airflow/Chart.yaml`:
```yaml
apiVersion: v2
name: airflow
description: A Helm chart for Kubernetes airflow
type: application
version: 0.1.0
dependencies:
  - name: airflow
    alias: airflow
    version: 1.15.0
    repository: https://airflow.apache.org/
  - name: postgresql-ha
    alias: postgres-ha
    version: 14.2.33
    repository: https://charts.bitnami.com/bitnami
```

Write the following to `charts/airflow/values.yaml`:
```yaml
airflow:
  dags:
      gitSync:
        enabled: true
        repo: https://github.com/Courtcircuits/airflow.git  # A small DAGS pipeline as an example.
        branch: main
        rev: HEAD
        subPath: "dags"
      persistence:
        enabled: false
  executor: "KubernetesExecutor"
  fernetkeysecretname: fernetkeysecret
  webserverSecretKeySecretName: webserversecret
  createUserJob:
    useHelmHooks: false
    applyCustomEnv: false
  migrateDatabaseJob:
    useHelmHooks: false
    applyCustomEnv: false
    jobAnnotations:
      "argocd.argoproj.io/hook": Sync
  postgresql:  # Default postgres deployed by the chart.
    enabled: false
  data:
    metadataConnection:
      user: postgres
      pass: postgres
      protocol: postgresql
      host: airflow-postgres-ha-postgresql.argocd.svc.cluster.local
      port: 5432
      db: "airflow"
global:
  postgresql:
    database: "airflow"
    username: "postgres"
    password: "postgres"
    repmgrUsername: "repmgr"
    repmgrPassword: "repmgr_password"
    postgresqlPassword: "postgres"
    postgresqlDatabase: "airflow"
  pgpool:
    adminPassword: "postgres"
    adminUsername: "admin"
postgres-ha: # HA Postgres
  postgresql:
    postgresPassword: "postgres"
  pgpool:
    adminPassword: "postgres"
```

Let's push these changes first:
```bash
git add charts/airflow/
git commit -m "ops(helm charts): add airflow chart"
git push
```
Then, we create an Application manifest for Airflow, that uses the Airflow Helm chart.
Write the following to `charts/root-app/templates/airflow.yaml`:
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: airflow
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/theotchlx/k3s-cluster.git
    path: charts/airflow/
    targetRevision: HEAD
    helm:
      valueFiles:
      - values.yaml
  destination:
    server: https://kubernetes.default.svc
    namespace: argocd
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

Let's push our changes:
```bash
git add charts/root-app/
git commit -m "ops(app of apps): add airflow app"
git push
```

