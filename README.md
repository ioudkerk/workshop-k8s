# workshop-k8s

## Create the cluster

```
cat <<EOF >> workshop-k3s.conf
---
hetzner_token: rlFGFHbo80H9eYoIsh6Ak3mF9mrE2aSa2UuZYnrWIwixbJaHm8BtNq3NXsKTr6tC
cluster_name: workshop
kubeconfig_path: "./kubeconfig"
k3s_version: v1.26.4+k3s1
public_ssh_key_path: "~/.ssh/id_rsa.pub"
private_ssh_key_path: "~/.ssh/id_rsa"
use_ssh_agent: false 
ssh_allowed_networks:
  - 0.0.0.0/0 # ensure your current IP is included in the range
api_allowed_networks:
  - 0.0.0.0/0 # ensure your current IP is included in the range
private_network_subnet: 10.0.0.0/16 # ensure this doesn't overlap with other networks in the same project
disable_flannel: false # set to true if you want to install a different CNI
schedule_workloads_on_masters: false
cloud_controller_manager_manifest_url: "https://github.com/hetznercloud/hcloud-cloud-controller-manager/releases/download/v1.18.0/ccm-networks.yaml"
csi_driver_manifest_url: "https://raw.githubusercontent.com/hetznercloud/csi-driver/v2.5.1/deploy/kubernetes/hcloud-csi.yml"
system_upgrade_controller_manifest_url: "https://raw.githubusercontent.com/rancher/system-upgrade-controller/master/manifests/system-upgrade-controller.yaml"
masters_pool:
  instance_type: cpx21
  instance_count: 1
  location: nbg1
worker_node_pools:
- name: small-static
  instance_type: cpx21
  instance_count: 3
  location: hel1
EOF

hetzner-k3s create --config workshop-k3s.conf
```

## Delegate DNS
Then we need to delegate the DNS to CloudFlare

Configure CloudFlare SSL Full Strict

## Create GitHub Repo

In this repo we will have all the deployment config. Because we want to be GitOps (git is the source of truth) we use ArgoCD and we will manage all the deployments with Helm.

The repo has the following architecture:

- hetzner -> in this folder we have the config to create the k3s cluster
- argocd -> here we have the config to deploy argocd
- helm-charts -> here we have all the helms that we deploy
- sealedSecrets -> here we have the sealed-secrets deployment config
- applications -> here we have the argocd applications definition
- rbac -> we will define here the rbac to access the cluster

We will need a Personal Access Token to allow ArgoCD read and push to the repo. 

1. Click on your username (upper right)
2. Click on Settings
3. Click on "Developers Settings"
4. Click on "Personal Access Token" -> Fine-grained tokens
   1. put a name to the token
   2. put 1 year of expiration
   3. choose "Only select repositories"
   4. Select "Contents" inside "Repository permissions"
5. Copy the token, it must be something like this: github_pat_11ABM33OY........NNg

## Configure the BluePrint

1. Install sealed-secrets

the first thing we need is the secret manager, we choose sealed-secrets to be 100% GitOps. 

run the following code to deploy the controller:

```
cd sealedSecrets
helm dependency build .
helm upgrade -i --dependency-update -n platform sealed-secrets . --create-namespace
```

It is installed as a dependenci of a chart, this strategy is known as Umbrella Helm. With this strategy we ensure to use always the same helm chart repo and version.

2. Install ArgoCD

Because we want to be 100% GitOps, we want to configure everying from git. So all our configurations must be commited to git, and that can be a security problem if we don't do it right.

With sealed secrets we can encrypt the secrets and commit them to git and be safe. So the first thing we need to do is create a secret to configure our private repositorie in ArgoCD.

First we need to create a file with the password:

```
mkdir ./tmp/k8s_secret
cd ./tmp/k8s_secret
```

```
touch password
echo https://github.com/ioudkerk/workshop-k8s.git > url
echo default > project
echo git > type
echo empty > username
```

Open the "password" file and paste there the GitHub Token

Now we are going to create a k8s secret template and then encrypt all the file:

```
SECRET_NAME=repo-workshop-k8s
kubectl create secret generic ${SECRET_NAME} \
    --from-file=password \
    --from-file=url \
    --from-file=project \
    --from-file=type \
    --from-file=username \
    --dry-run=client \
    --output yaml > repo_secret.yaml
```

Now we will encrypt the secret:

```
kubeseal < repo_secret.yaml --controller-namespace platform --controller-name sealed-secrets --scope=namespace-wide --namespace argocd -o yaml > repo-k8s-workshop.yaml
```

> NOTES: We use the namespace-wide scope, so this secret can be decrypted only in that namespace, and we use the argocd namespace.
> 
> NOTES: The kubeseal needs to know in wich namespace is deployed the sealed-secrets controller and the sealed-secrets service name to be able to use it. In our case it is **platform** namespace and the service name is **sealed-secret**

Now we need to move the encrypted secret to the ArgoCD template folder and remove all this files:

```
mv repo-k8s-workshop.yaml ${HOME_PROJECT}/argocd/templates/
```