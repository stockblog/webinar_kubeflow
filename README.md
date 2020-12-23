# Webinar_kubeflow

### We will install and test kubeflow in Mail.ru Cloud Solutions
#### Tested with Kubeflow 1.1, Kubernetes 1.16.4, Istio 1.3.1, Client VM Ubuntu 18.04 

## Kubeflow links

### Overview
https://www.kubeflow.org/docs/started/kubeflow-overview/

### Minimum system requirements
https://www.kubeflow.org/docs/started/k8s/overview/

### Multi user instruction
https://www.kubeflow.org/docs/started/k8s/kfctl-istio-dex/



## Part 1. Kubeflow installation
### Prerequisites


### Create K8s cluster in mcs and download kubeconfig
Instruction: https://mcs.mail.ru/help/kubernetes/clusterfast

Kubernetes as a Service: https://mcs.mail.ru/app/services/containers/add/

### Install kubectl
https://mcs.mail.ru/help/ru_RU/k8s-start/connect-k8s

### Set path to kubeconfig for kubectl
```console
export KUBECONFIG=/replace_with_path/to_your_kubeconfig.yaml
```

### also it will be easier to work with kubectl while enabling autocomplete and using alias
```console
alias k=kubectl
source <(kubectl completion bash)
complete -F __start_kubectl k
```


### Configure K8s cluster
### We will need this feature in K8s to start Kubeflow
https://kubernetes.io/docs/tasks/configure-pod-container/configure-service-account/#service-account-token-volume-projection
#### 1. Assign white ip to master node of K8s cluster
#### 2. SSH to master 
```console
ssh -i your_key.pem centos@ip_of_master
```
#### 3. Edit /etc/kubernetes/apiserver
```console
sudo vim /etc/kubernetes/apiserver
```
#### add to KUBE_API_ARGS
```console
--service-account-issuer=kubernetes.default.svc  --service-account-signing-key-file=/etc/kubernetes/certs/ca.key --service-account-api-audiences=api,istio-ca
```
#### 4. Restart cluster from UI



### Istio install
```console
curl -L https://istio.io/downloadIstio | ISTIO_VERSION=1.3.1 TARGET_ARCH=x86_64 sh -
```

#### 1. To configure the istioctl client tool for your workstation, add the /home/ubuntu/istio/istio-1.3.1/bin directory to your environment path variable with:
```console
export PATH="$PATH:/home/ubuntu/istio/istio-1.3.1/bin"
```

#### 2. Begin the Istio pre-installation check by running:
```console
istioctl verify-install
```
```console
cd ~/istio/istio-1.3.1
```

#### 3. Install all the Istio Custom Resource Definitions (CRDs)
```console
for i in install/kubernetes/helm/istio-init/files/crd*yaml; do kubectl apply -f $i; done
```
#### 4. install demo profile
```console
kubectl apply -f install/kubernetes/istio-demo.yaml
```

#### 5. check istio
```console
kubectl get pods -n istio-system
```

### Kubeflow CLI install
#### 1. Download the kfctl v1.1.0 release from the Kubeflow releases page.
#### https://github.com/kubeflow/kfctl/releases/tag/v1.1.0


#### 2. Unpack the tar ball:
```console
tar -xvf kfctl_<release tag>_<platform>.tar.gz
```

#### 3. Create env variables
```console
# Add kfctl to PATH, to make the kfctl binary easier to use.
# Use only alphanumeric characters or - in the directory name.
export PATH=$PATH:"~/"

# Set the following kfctl configuration file:
export CONFIG_URI="https://raw.githubusercontent.com/kubeflow/manifests/v1.1-branch/kfdef/kfctl_istio_dex.v1.1.0.yaml"

# Set KF_NAME to the name of your Kubeflow deployment. You also use this
# value as directory name when creating your configuration directory.
# For example, your deployment name can be 'my-kubeflow' or 'kf-test'.
export KF_NAME=kubeflow-mcs

# Set the path to the base directory where you want to store one or more 
# Kubeflow deployments. For example, /opt.
# Then set the Kubeflow application directory for this deployment.
export BASE_DIR=~/
export KF_DIR=${BASE_DIR}/${KF_NAME}
```

### Set up configuration and deploy Kubeflow
```console
mkdir -p ${KF_DIR}
cd ${KF_DIR}
kfctl build -V -f ${CONFIG_URI}
export CONFIG_FILE=${KF_DIR}/kfctl_istio_dex.v1.1.0.yaml
```

#### 1. Edit the configuration files. 
#### We need to delete part with Istio as we already have installed it.
#### Additional information https://www.kubeflow.org/docs/other-guides/kustomize/

```console
nano $CONFIG_FILE

#delete this part	
  - kustomizeConfig:
      repoRef:
        name: manifests
        path: stacks/kubernetes/application/istio-1-3-1-stack
    name: istio-stack
	
#and this	
	  - kustomizeConfig:
      repoRef:
        name: manifests
        path: istio/istio/base
    name: istio
	

```

#### 2. Run the kfctl apply command to deploy Kubeflow:
```console
kfctl apply -V -f ${CONFIG_FILE}
```



### Add static users for basic auth 
To add users to basic auth, you just have to edit the Dex ConfigMap under the key staticPasswords.

#### 1. Download the dex config
```console
kubectl get configmap dex -n auth -o jsonpath='{.data.config\.yaml}' > dex-config.yaml
```

#### 2. Edit the dex config with extra users.
The password must be hashed with bcrypt with an at least 10 difficulty level.
You can use an online tool like: https://passwordhashing.com/BCrypt
```console 
nano dex-config.yaml
```

#### 3. After editing the config, update the ConfigMap
```console
kubectl create configmap dex --from-file=config.yaml=dex-config.yaml -n auth --dry-run -oyaml | kubectl apply -f -
```

#### 4. Restart Dex to pick up the changes in the ConfigMap
```console
kubectl rollout restart deployment dex -n auth
```


### Expose Kubeflow 
#### 1. Secure with HTTPS 
```console
cat <<EOF | kubectl apply -f -
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  annotations:
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"networking.istio.io/v1alpha3","kind":"Gateway","metadata":{"annotations":{},"name":"kubeflow-gateway","namespace":"kubeflow"},"spec":{"selector":{"istio":"ingressgateway"},"servers":[{"hosts":["*"],"port":{"name":"h$  creationTimestamp: "2020-11-11T07:34:04Z"
  generation: 1
  name: kubeflow-gateway
  namespace: kubeflow
spec:
  selector:
    istio: ingressgateway
  servers:
  - hosts:
    - '*'
    port:
      name: http
      number: 80
      protocol: HTTP
    tls:
      httpsRedirect: true
  - hosts:
    - '*'
    port:
      name: https
      number: 443
      protocol: HTTPS
    tls:
      mode: SIMPLE
      privateKey: /etc/istio/ingressgateway-certs/tls.key
      serverCertificate: /etc/istio/ingressgateway-certs/tls.crt
EOF
```



#### 2. Some magic. 
```console
kubectl patch service -n istio-system istio-ingressgateway -p '{"spec": {"type": "NodePort"}}'

#wait one minute then 
kubectl patch service -n istio-system istio-ingressgateway -p '{"spec": {"type": "LoadBalancer"}}'

#wait for assigning external IP
```

#### 3. Get External IP of istio-ingressgateway
```console
#you can do it with next command, then look for External IP column
kubectl get svc -n istio-system

#or you can get IP with that command
kubectl get svc -n istio-system istio-ingressgateway -o jsonpath='{.status.loadBalancer.ingress[0]}'
```

#### 4. Create the Certificate with cert-manager:
```console
export INGRESS_IP=INSERT_IP_RECEIVED_ON_PREV_STEP

cat <<EOF | kubectl apply -f -
apiVersion: cert-manager.io/v1alpha2
kind: Certificate
metadata:
  name: istio-ingressgateway-certs
  namespace: istio-system
spec:
  commonName: istio-ingressgateway.istio-system.svc
  # Use ipAddresses if your LoadBalancer issues an IP
  ipAddresses:
  - ${INGRESS_IP}
  isCA: true
  issuerRef:
    kind: ClusterIssuer
    name: kubeflow-self-signing-issuer
  secretName: istio-ingressgateway-certs
EOF
```


### 5. Then Navigate to https://LoadBalancer IP/ and start using Kubeflow
You need to create namespace at first login. 


## Part 2. Practice with Kubeflow

### Usefull links
https://github.com/kubeflow/examples/tree/master/mnist#vanilla

### Prerequisites
#### Configure docker credentials
Why do we need this?
Kaniko is used by fairing to build the model every time the notebook is run and deploy a fresh model. The newly built image is pushed into the DOCKER_REGISTRY and pulled from there by subsequent resources.

#### Get your docker registry user and password encoded in base64
```console
echo -n USER:PASSWORD | base64
```

#### Then Create a config.json file with your Docker registry url and the previous generated base64 string
```console
nano config.json

#then paste this
{
	"auths": {
		"https://index.docker.io/v1/": {
			"auth": "generated_base64_string"
		}
	}
}
```

Your config.json should look like this example:
```console
{
        "auths": {
                "https://index.docker.io/v1/": {
                        "auth": "fdwNzY2xvdWQ6O/sdggfd439Fiwt"
                }
        }
}
```

#### Create a config-map in the namespace you're using with the docker config
Namespace name is the same you created at first login in Kubeflow 
```console
kubectl create --namespace ${NAMESPACE} configmap docker-config --from-file=path_to_config.json
```


### Launch a Jupyter Notebook in Kubeflow
#### This can be done with "Notebook Servers" in the Kubeflow menu

The tutorial is run on Jupyter Tensorflow 1.15 image.
#### Launch a terminal in Jupyter and clone the kubeflow/examples repo
```console
git clone https://github.com/stockblog/kubeflow_examples.git git_kubeflow_examples
```

#### Open the notebook git_kubeflow_examples/mnist/mnist_mcs_k8s.ipynb
Follow the notebook to train and deploy on MNIST on Kubeflow


### You will need MINIO service endpoint
```console
kubectl get svc minio-service -n kubeflow
```

### Also, external ip of istio-ingressgateway
```console
kubectl get svc istio-ingressgateway -n istio-system
```

### If you want to deploy tensorboard, you need to give permissions to create virtualservices and create rolebinding	
```console	   
cat <<EOF | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: create-virtualservices-custom
  labels:
    rbac.authorization.kubeflow.org/aggregate-to-kubeflow-edit: "true"
rules:
- apiGroups: ["networking.istio.io"]
  resources: ["virtualservices"]
  verbs: ["get", "list", "watch", "create", "delete", "update", "patch"]
EOF
```

```console
export USER_NAMESPACE_KF=REPLACE_WITH_YOUR_NAMESPACE

kubectl create --namespace=istio-system rolebinding \
--clusterrole=kubeflow-view \
--serviceaccount=${USER_NAMESPACE_KF}:default-editor \
 ${USER_NAMESPACE_KF}-ingressgateway-view
```


### Simple Pipeline Run

Envoy filter to inject the kubeflow-userid header from notebook to ml-pipeline service. 

additional info

https://www.kubeflow.org/docs/pipelines/sdk/sdk-overview/

https://github.com/kubeflow/pipelines/issues/4440

```
#REPLACE WITH YOUR PARAMETERS

export USER_NAMESPACE_KF=admin-kubeflow
export NOTEBOOK=mnist-test
export USER=admin@kubeflow.org
```

```console
cat <<EOF | kubectl apply -f -
apiVersion: rbac.istio.io/v1alpha1
kind: ServiceRoleBinding
metadata:
  name: bind-ml-pipeline-nb-${USER_NAMESPACE_KF}
  namespace: kubeflow
spec:
  roleRef:
    kind: ServiceRole
    name: ml-pipeline-services
  subjects:
  - properties:
      source.principal: cluster.local/ns/${USER_NAMESPACE_KF}/sa/default-editor
EOF
```	  
	  

```console
cat <<EOF | kubectl apply -f -
apiVersion: networking.istio.io/v1alpha3
kind: EnvoyFilter
metadata:
  name: add-header
  namespace: ${USER_NAMESPACE_KF}
spec:
  configPatches:
  - applyTo: VIRTUAL_HOST
    match:
      context: SIDECAR_OUTBOUND
      routeConfiguration:
        vhost:
          name: ml-pipeline.kubeflow.svc.cluster.local:8888
          route:
            name: default
    patch:
      operation: MERGE
      value:
        request_headers_to_add:
        - append: true
          header:
            key: kubeflow-userid
            value: ${USER}
  workloadSelector:
    labels:
      notebook-name: ${NOTEBOOK}
EOF
```

#### Open the notebook git_kubeflow_examples/pipelines/simple-notebook-pipeline/Simple Notebook Pipeline.ipynb

