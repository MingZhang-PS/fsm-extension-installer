## Kyma admin to setup FSM-Extension-Installer related components 
(We need to make the whole section to be a single Shell scripts for better automation!)

### Config the kubectl and Helm
- Get the kubeconfig file and configure the CLI according to [get the kubeconfig file and configure the cli](https://kyma-project.io/docs/#tutorials-sample-service-deployment-on-a-cluster-get-the-kubeconfig-file-and-configure-the-cli)
- Configure the helm client according to [installation use helm](https://kyma-project.io/docs/#installation-use-helm)

### Create a dedicated namespace for installer on Kyma

- Create namespace for installer:
```kubectl create ns <your installer namespace>```
- Switch to the new namespace:
```kubectl config set-context --current --namespace=<your installer namespace>```
- Check if you can do everything in this new namespace
```kubectl auth can-i '*' '*'```

### Install the helm-operator on Kyma
- Add the fluxcd repo:
```helm repo add fluxcd https://charts.fluxcd.io```
- Install the HelmRelease CRD:
```kubectl apply -f https://raw.githubusercontent.com/fluxcd/helm-operator/master/deploy/crds.yaml```
- Install Helm Operator for Helm v3 only (I don't try with v2 yet):
```yaml
helm upgrade -i helm-operator fluxcd/helm-operator \
--namespace <your installer namespace> \
--set configureRepositories.enable=true \
--set 'configureRepositories.repositories[0].name=stable' \
--set 'configureRepositories.repositories[0].url=https://kubernetes-charts.storage.googleapis.com' \
--set 'configureRepositories.repositories[1].name=podinfo' \
--set 'configureRepositories.repositories[1].url=https://stefanprodan.github.io/podinfo' \
--set helm.versions=v3 \
--tls
```
- Verify if the Helm Operator works. If everything ok, you will find a pod named "podinfo" created successfully
	```yaml
	apiVersion: helm.fluxcd.io/v1
	kind: HelmRelease
	metadata:
	  name: podinfo
	spec:
	  releaseName: podinfo
	  chart:
		repository: https://stefanprodan.github.io/podinfo
		version: 2.1.0
		name: podinfo
	  values:
		replicaCount: 1
	```

### Install the FSM-Extension-Installer on Kyma
- Create the dedicated role for the new service account
	```yaml
	apiVersion: rbac.authorization.k8s.io/v1
	kind: ClusterRole
	metadata:
	  name: fsm-extension-installer-role
	rules:
	- apiGroups: ["helm.fluxcd.io"]
	  resources: ["helmreleases"]
	  verbs: ["*"]
	- apiGroups: ["gateway.kyma-project.io"]
	  resources: ["apis"]
	  verbs: ["get"]
	```

- Create the clusterrolebinding
	```yaml
	apiVersion: rbac.authorization.k8s.io/v1
	kind: ClusterRoleBinding
	metadata:
	  name: fsm-extension-installer-role-binding
	roleRef:
	  apiGroup: rbac.authorization.k8s.io
	  kind: ClusterRole
	  name: fsm-extension-installer-role
	subjects:
	- kind: ServiceAccount
	  name: default
	  namespace: <your installer namespace>
	```

- use kubectl to install FSM-Extension-Installer, see [lambda.yaml](./lambda.yaml) and [api.yaml](./api.yaml)

### Reference
####Helm Operator 
https://github.com/fluxcd/helm-operator/blob/master/chart/helm-operator/README.md
https://docs.fluxcd.io/projects/helm-operator/en/1.0.0-rc9/references/operator.html

####HelmRelease CRD
https://docs.fluxcd.io/projects/helm-operator/en/1.0.0-rc9/references/helmrelease-custom-resource.html

####Chart Repository
https://medium.com/containerum/how-to-make-and-share-your-own-helm-package-50ae40f6c221
https://whmzsu.github.io/helm-doc-zh-cn/chart/chart_repository-zh_cn.html
https://www.baeldung.com/kubernetes-helm