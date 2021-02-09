Kubernetes and Magnum
=====================
JupyterHub can be deployed to a Kubernetes cluster provisioned using OpenStack Magnum. 

The main steps are:

# Provision a new Kubernetes cluster via OpenStack Magnum
# Configure external access
# Install helm
# Install and configure JupyterHub
# Dask

# Provision new cluster
To provision a new cluster, first log into the Horizon interface on the IRIS cloud that you are configuring.
Check that you have selected the correct project under which to create the cluster, by selected the project in menu at the top left of the web page.
Click on "Container Infra" to list the available cluster templates.
While this will vary from system to system, generally it is best to select the most recent template.
Click "Create Cluster" to start setting the cluster parameters.
Navigate through the options, ensuring that "Docker Volume Size" is set at at least 10GB; the default size is too small for practical use. In "Keypair", select the keypair that you will use to ssh to the created cluster.
Create 1 master instance and at least one node instance.
Once all required parameters have been supplied, click "Submit" to begin cluster creation.
Cluster creation may take some time. Clusters can be viewed by navigating to "Container Infra->Clusters"
Once creation is complete, the compute instances within the cluster can be listed by navigating to "Compute"->"Instances"
The master node can be identified by a name <..>-master-0 or similar. 


# Configure external access
To log into the master node via ssh, you will need to allocate an external floating point ip address.
This may differ from system to system, depending on the local configuration of the firewall; you may need to contact your system support to request that the firewall be opened for ingress on the address that you have allocated to your master node.

# Install helm
Confirm that kubernetes is correctly installed by executing "kubectl --help"
Install helm by following the instructions at https://zero-to-jupyterhub.readthedocs.io/en/latest/kubernetes/setup-helm.html (the simplest way is to execute 'curl https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3 | bash')
Confirm that helm is correctly installed by executing 'helm list'

# Install and configure JupyterHub
To install JupyterHub, follow the instructions at https://zero-to-jupyterhub.readthedocs.io/en/latest/jupyterhub/index.html
Note that JupyterHub is sensitive to differences in versions between the installed hub component of JupyterHub and the version of the hub component used to build the docker image specified in the config.yaml file. The following config file was successfully used for installation using helm chart version=0.9.0 and a compatible version of the jupyter datascience-notebook image 

proxy:
  secretToken: <insert your value>
hub:
  db:
    type: sqlite-memory
singleuser:
  image:
    name: jupyter/datascience-notebook
    tag: hub-1.1.0
  storage:
    type: sqlite-memory

kubectl can then be used to verify that JupyterHub is running and accessible:
kubectl get service --namespace jhub
NAME           TYPE           CLUSTER-IP       EXTERNAL-IP       PORT(S)                      AGE
hub            ClusterIP      10.254.174.51    <none>            8081/TCP                     112s
proxy-api      ClusterIP      10.254.104.162   <none>            8001/TCP                     112s
proxy-public   LoadBalancer   10.254.102.185   128.232.227.148   443:31962/TCP,80:30774/TCP   112s

Note that depending on how the firewall is configured for your system, JupyterHub may not be able to automatically acquire an external ip address. If this is the case, then you will need to manually allocate a new floating ip address via Horizon, and request that this address be opened for ingress. This address can then be specified on the config file as follows, and jupyterhub redeployed using helm

proxy:
  service:
    loadBalancerIP: <your address here>

You should then be able to browse to the external address shown for the LoadBalancer, and start to create notebooks.

# Dask
Dask is a powerful set of libraries that can provide parallel processing and automated scaling accessed from a python notebook. dask-gateway provides a set of services that can be easily installed onto kubernetes using helm.
Follow the instructions at https://gateway.dask.org/install-kube.html#install-dask-gateway, using the default config file at https://github.com/dask/dask-gateway/blob/master/resources/helm/dask-gateway/values.yaml
Dask will automatically try to acquire an external floating ip address for it's load balancer. This can be listed using kubectl, eg:

kubectl get service --namespace dask-gateway
NAME                   TYPE           CLUSTER-IP      EXTERNAL-IP       PORT(S)        AGE
api-dask-gateway       ClusterIP      10.254.61.172   <none>            8000/TCP       3h55m
traefik-dask-gateway   LoadBalancer   10.254.138.19   128.232.227.222   80:31454/TCP   3h55m

If your firewall does not automatically allow ingress for new floating ip addresses, you will need to contact your cloud support for advice on how to obtain a new address and request ingress. You will then need to reinstall dask specifying your new floating ip address as the address of the load balancer.

Your dask installation can be tested by creating a new Jupyter notebook, then running a simple dask task from within it.
Note that dask is sensitive to mismatches in versions between libraries on the worker images and in the calling client.
The dask_gateway client provides a get_versions method which checks for any potential mismatches.

The following code provides a simple test case for running in a notebook:

pip install dask_gateway
from dask_gateway import Gateway
import dask.array as da
try:
  gateway = Gateway("address of your dask load balancer")
  cluster = gateway.new_cluster()
  cluster.scale(5)
  client = cluster.get_client()
  print('created cluster, allocating random array')
  a = da.random.normal(size=(10000, 10000), chunks=(500, 500))
  print('starting calculation')
  print('mean {0}', a.mean().compute())
  print('shutting down cluster')
  cluster.shutdown()
  print('done')
except Exception as e:
  print(e)

execute this, and via kubectl you should be able to see dask starting worker pods on demand, eg:

kubectl get pods --namespace dask-gateway
NAME                                                 READY   STATUS    RESTARTS   AGE
api-dask-gateway-86f78b7bf-8knfn                     1/1     Running   0          4h36m
controller-dask-gateway-775b47fffc-bmq77             1/1     Running   0          4h36m
dask-scheduler-a21b3dcd471c402ab3e53a8eac625a5e      1/1     Running   0          47s
dask-worker-a21b3dcd471c402ab3e53a8eac625a5e-4k8nb   1/1     Running   0          39s
dask-worker-a21b3dcd471c402ab3e53a8eac625a5e-4xnml   1/1     Running   0          39s
dask-worker-a21b3dcd471c402ab3e53a8eac625a5e-6pf5g   1/1     Running   0          39s
dask-worker-a21b3dcd471c402ab3e53a8eac625a5e-94z2w   0/1     Pending   0          39s
dask-worker-a21b3dcd471c402ab3e53a8eac625a5e-vm2sp   1/1     Running   0          39s
traefik-dask-gateway-754b78c8-fqcx6                  1/1     Running   0          4h36m
