# GPU Aware Scheduling
GPU Aware Scheduling (GAS) allows using GPU resources such as memory amount for scheduling decisions in Kubernetes. It is used to optimize scheduling decisions when the POD resource requirements include the use of several GPUS or fragments of GPUs on a node, instead of traditionally mapping a GPU to a pod.

GPU Aware Scheduling is deployed in a single pod on a Kubernetes Cluster. 

**This software is a pre-production alpha version and should not be deployed to production servers.**

### GPU Aware Scheduler Extender
GPU Aware Scheduler Extender is contacted by the generic Kubernetes Scheduler every time it needs to make a scheduling decision for a pod which requests the `gpu.intel.com/i915` resource.
The extender checks if there are other `gpu.intel.com/`-prefixed resources associated with the workload.
If so, it inspects the respective node resources and filters out those nodes where the POD will not really fit.
This is implemented and configured as a [Kubernetes Scheduler Extender.](https://kubernetes.io/docs/concepts/configuration/manage-compute-resources-container/#cluster-level-extended-resources)

The typical use-case which GAS solves can be described with the following imaginary setup.
1) Node has two GPUs, each having 8GB of GPU memory. The node advertises 16GB of GPU memory as a kubernetes extended resource.
2) POD instances need 5GB of GPU memory each
3) A replicaset of 3 is created for said PODs, totaling a need for 3*5GB = 15GB of GPU memory.
4) Kubernetes Scheduler, if left to its own decision making, can place all the PODs on this one node with only 2 GPUs, since it only considers the memory amount of 16GB. However to be able to do such a deployment and run the PODs successfully, the last instance would need to allocate the GPU memory from two of the GPUs.
5) GAS solves this issue by keeping book of the individual GPU memory amount. After two PODs have been deployed on the node, both GPUs have 3GB of free memory left. When GAS sees that the need for memory is 5GB but none of the GPUs in that node have as much (even though combined there is still 6GB left) it will filter out that node from the list of the nodes k8s scheduler proposed. The POD will not be deployed on that node.

GAS tries to be agnostic about resource types. It doesn't try to have an understanding of the meaning of the resources, they are just numbers to it, which it identifies from other Kubernetes extended resources with the prefix "gpu.intel.com/". The only resource treated differently is the GPU-plugin "i915"-resource, which is considered to describe "from how many GPUs the GPU-resources for the POD should be evenly consumed". That is, if each GPU has e.g. capacity of 1000 "gpu.intel.com/millicores", and POD spec has a limit for two (2) "gpu.intel.com/i915" and 2000 "gpu.intel.com/millicores", that POD will consume 1000 millicores from two GPUs, totaling 2000 millicores. After GAS has calculated the resource requirement per GPU by dividing the extended resource numbers with the number of requested "i915", deploying the POD to a node is only allowed if there are enough resources in the node to satisfy fully the per-GPU resource requirement in as many GPUs as requested in "i915" resource. Typical PODs use just one i915 and consume resources from only a single GPU.

GAS heavily utilizes annotations. It itself annotates PODs after making filtering decisions on them, with a precise timestamp at annotation named "gas-ts". The timestamp can then be used for figuring out the time-order of the GAS-made scheduling decision for example during the GPU-plugin resource allocation phase, if the GPU-plugin wants to know the order of GPU-resource consuming POD deploying inside the node. Another annotation which GAS adds is "gas-container-cards". It will have the names of the cards selected for the containers. Containers are separated by "|", and card names are separated by ",". Thus a two-container POD in which both containers use 2 GPUs, could get an annotation "card0,card1|card2,card3". These annotations are then consumed by the Intel GPU device plugin.

GAS also expects labels to be in place for the nodes, in order to be able to keep book of the cluster GPU resource status. Nodes with GPUs shall be labeled with label name "gpu.intel.com/cards" and value shall be in form "card0.card1.card2.card3"... where the card names match with the intel GPUs which are currently found under /sys/class/drm folder, and the dot serves as separator. GAS expects all GPUs of the same node to be homogeneous in their resource capacity, and calculates the GPU extended resource capacity as evenly distributed to the GPUs listed by that label.


## Usage with NFD and the GPU-plugin
A worked example for GAS is available [here](docs/usage.md)

### Quick set up
The deploy folder has all of the yaml files necessary to get GPU Aware Scheduling running in a Kubernetes cluster. Some additional steps are required to configure the generic scheduler.

#### Extender configuration
Note: a shell script that shows these steps can be found [here](deploy/extender-configuration). This script should be seen as a guide only, and will not work on most Kubernetes installations.

The extender configuration files can be found under deploy/extender-configuration.
GAS Scheduler Extender needs to be registered with the Kubernetes Scheduler. In order to do this a configmap should be created like the below:
```
apiVersion: v1alpha1
kind: ConfigMap
metadata:
  name: scheduler-extender-policy
  namespace: kube-system
data:
  policy.cfg: |
    {
        "kind" : "Policy",
        "apiVersion" : "v1",
        "extenders" : [
            {
              "urlPrefix": "https://gpu-service.default.svc.cluster.local:9001",
              "apiVersion": "v1",
              "filterVerb": "scheduler/filter",
              "bindVerb": "scheduler/bind",
              "weight": 1,
              "enableHttps": true,
              "managedResources": [
                   {
                     "name": "gpu.intel.com/i915",
                     "ignoredByScheduler": false
                   }
              ],
              "ignorable": true,
              "nodeCacheCapable": true
              "tlsConfig": {
                     "insecure": false,
                     "certFile": "/host/certs/client.crt",
                     "keyFile" : "/host/certs/client.key"
              }
          }
         ]
    }

```

A similar file can be found [in the deploy folder](./deploy/extender-configuration/scheduler-extender-configmap.yaml). This configmap can be created with ``kubectl apply -f ./deploy/scheduler-extender-configmap.yaml``
The scheduler requires flags passed to it in order to know the location of this config map. The flags are:
```
    - --policy-configmap=scheduler-extender-policy
    - --policy-configmap-namespace=kube-system
```

If scheduler is running as a service these can be added as flags to the binary. If scheduler is running as a container - as in kubeadm - these args can be passed in the deployment file.
Note: For Kubeadm set ups some additional steps may be needed.
1) Add the ability to get configmaps to the kubeadm scheduler config map. (A cluster role binding for this is at deploy/extender-configuration/configmap-getter.yaml)
2) Add the ``dnsPolicy: ClusterFirstWithHostNet`` in order to access the scheduler extender by service name.

After these steps the scheduler extender should be registered with the Kubernetes Scheduler.

#### Deploy GAS
GPU Aware Scheduling uses go modules. It requires Go 1.13+ with modules enabled in order to build. GAS has been tested with Kubernetes 1.15+.
A yaml file for GAS is contained in the deploy folder along with its service and RBAC roles and permissions.

**Note:** If run without the unsafe flag a secret called extender-secret will need to be created with the cert and key for the TLS endpoint.
GAS will not deploy if there is no secret available with the given deployment file.

A secret can be created with:

``
kubectl create secret tls extender-secret --cert /etc/kubernetes/<PATH_TO_CERT> --key /etc/kubernetes/<PATH_TO_KEY> 
``
In order to build in your host:

``make build``

You can also build inside docker, which creates the container:

``make image``

The "deploy"-folder has the necessary scripts for deploying. You can simply deploy by running:

``kubectl apply -f deploy/``

After this is run GAS should be operable in the cluster and should be visible after running ``kubectl get pods``

Remember to run the configure-scheduler.sh script, or perform similar actions in your cluster if the script does not work in your environment directly.

### Configuration flags
The below flags can be passed to the binaries at run time.

#### GAS Scheduler Extender
name |type | description| usage | default|
-----|------|-----|-------|-----|
|kubeConfig| string |location of kubernetes configuration file | --kubeConfig /root/filename|~/.kube/config
|port| int | port number on which the scheduler extender will listen| --port 32000 | 9001
|cert| string | location of the cert file for the TLS endpoint | --cert=/root/cert.txt| /etc/kubernetes/pki/ca.key
|key| string | location of the key file for the TLS endpoint| --key=/root/key.txt | /etc/kubernetes/pki/ca.key
|cacert| string | location of the ca certificate for the TLS endpoint| --key=/root/cacert.txt | /etc/kubernetes/pki/ca.crt
|unsafe| bool | whether or not to listen on a TLS endpoint with the scheduler extender | --unsafe=true| false

## Adding the resource to make a deployment use GAS Scheduler Extender

For example,  in a deployment file: 
```
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: demo-app
  labels:
    app: demo
spec:
  replicas: 1
  selector:
    matchLabels:
      app: demo 
  template:
    metadata:
      labels:
        app: demo
    spec:
      containers:
      - name: nginx
        image: nginx:latest
        imagePullPolicy: IfNotPresent
        resources:
          limits:
            gpu.intel.com/i915: 1
```

There is one change to the yaml here:
- A resources/limits entry requesting the resource gpu.intel.com/i915. This is used to restrict the use of GAS to only selected pods. If this is not in a pod spec the pod will not be scheduled by GAS.

### Unsupported use-cases

Topology Manager and GAS card selections can conflict. Using both at the same time is not supported. You may use topology manager without GAS.

Selecting deployment node directly in POD spec [nodeName](https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/#nodename) bypasses the scheduler and therefore also GAS. This is obviously a use-case which can't be supported by GAS, so don't use that mechanism, if you want to run the scheduler and GAS.

### Security
GAS Scheduler Extender is set up to use in-Cluster config in order to access the Kubernetes API Server. When deployed inside the cluster this along with RBAC controls configured in the installation guide, will give it access to the required resources.

Additionally GAS Scheduler Extender listens on a TLS endpoint which requires a cert and a key to be supplied.
These are passed to the executable using command line flags. In the provided deployment these certs are added in a Kubernetes secret which is mounted in the pod and passed as flags to the executable from there.

## Communication and contribution

Report a bug by [filing a new issue](https://github.com/intel/platform-aware-scheduling/issues).

Contribute by [opening a pull request](https://github.com/intel/platform-aware-scheduling/pulls).

Learn [about pull requests](https://help.github.com/articles/using-pull-requests/).

**Reporting a Potential Security Vulnerability:** If you have discovered potential security vulnerability in GAS, please send an e-mail to secure@intel.com. For issues related to Intel Products, please visit [Intel Security Center](https://security-center.intel.com).

It is important to include the following details:

- The projects and versions affected
- Detailed description of the vulnerability
- Information on known exploits

Vulnerability information is extremely sensitive. Please encrypt all security vulnerability reports using our [PGP key](https://www.intel.com/content/www/us/en/security-center/pgp-public-key.html).

A member of the Intel Product Security Team will review your e-mail and contact you to collaborate on resolving the issue. For more information on how Intel works to resolve security issues, see: [vulnerability handling guidelines](https://www.intel.com/content/www/us/en/security-center/vulnerability-handling-guidelines.html).

