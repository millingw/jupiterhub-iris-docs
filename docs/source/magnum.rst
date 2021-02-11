Kubernetes and Magnum
=====================
JupyterHub can be deployed to a Kubernetes cluster provisioned using OpenStack Magnum. 

The main steps are:

#. Provision a new Kubernetes cluster via OpenStack Magnum  
#. Configure external access
#. Install helm
#. Install and configure JupyterHub
#. Install and configure Dask or Dask

Provision new cluster
---------------------

To provision a new cluster, first log into the Horizon interface on the IRIS cloud that you are configuring. 

#. Check that you have selected the correct project under which to create the cluster, by selected the project in menu at the top left of the web page.
#. Click on "Container Infra" to list the available cluster templates. While this will vary from system to system, generally it is best to select the most recent template.
#. Click "Create Cluster" to start setting the cluster parameters.
    * Navigate through the options, ensuring that "Docker Volume Size" is set at at least 10GB; the default size is too small for practical use. 
    * In "Keypair", select the keypair that you will use to ssh to the created cluster.
    * Create 1 master instance and at least one node instance.
    * Once all required parameters have been supplied, click "Submit" to begin cluster creation.
    * Cluster creation may take some time. Clusters can be viewed by navigating to "Container Infra->Clusters"
    * Once creation is complete, the compute instances within the cluster can be listed by navigating to "Compute"->"Instances". 
    * The cluster master node can be identified by a name <..>-master-0 or similar. 

To log into the master node via ssh, you will need to allocate an external floating point ip address.
This may differ from system to system, depending on the local configuration of the firewall; you may need to contact your system support to request that the firewall be opened for ingress on the address that you have allocated to your master node.

Install helm
------------
Helm is one of the most popular tools for managing applications on kubernetes clusters. Applications are configured via helm charts; usually these contain a large number of default values, and only a simple additional configuration file is needed to override any specific settings.

ssh into your master node using the keypair that you specified during cluster creation

Confirm that kubernetes was correctly installed during the cluster setup::

    kubectl --help

Install helm by following the instructions at https://zero-to-jupyterhub.readthedocs.io/en/latest/kubernetes/setup-helm.html (the simplest way is to execute 'curl https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3 | bash')
Confirm that helm is correctly installed:: 

  helm --list

Install and configure JupyterHub
--------------------------------
To install JupyterHub, follow the instructions at https://zero-to-jupyterhub.readthedocs.io/en/latest/jupyterhub/index.html

JupyterHub provides many options for user environments and persistent storage. The simplest setup is to use an in-memory database ("sqlite-memory"), which means that user resources will not persist between sessions (when they log out, everything is wiped).

Note that JupyterHub is sensitive to differences in versions between the installed hub component of JupyterHub and the version of the hub component used to build the docker image specified in the config.yaml file. 
The following config file was successfully used for installation using helm chart version=0.9.0 and a compatible version of the jupyter datascience-notebook image:: 

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

kubectl can then be used to verify that JupyterHub is running and accessible (assuming that you specified jhub as the namespace during installation)::

  kubectl get service --namespace jhub
  NAME           TYPE           CLUSTER-IP       EXTERNAL-IP       PORT(S)                      AGE
  hub            ClusterIP      10.254.174.51    <none>            8081/TCP                     112s
  proxy-api      ClusterIP      10.254.104.162   <none>            8001/TCP                     112s
  proxy-public   LoadBalancer   10.254.102.185   128.232.227.148   443:31962/TCP,80:30774/TCP   112s

(Note that depending on how the firewall is configured for your system, JupyterHub may not be able to automatically acquire an external ip address, and the load balancer will show as "Pending" for its EXTERNAL-IP. If this is the case, then you will need to manually allocate a new floating ip address via Horizon, and request that this address be opened for ingress.) 

This address can then be specified on the config file as follows, and jupyterhub redeployed using helm as per the zero-to-jupyterhub documentation::

  proxy:
    service:
      loadBalancerIP: <your external IP address here>

You should then be able to browse to the external address shown for the LoadBalancer, and start to create notebooks.

Dask
----
Dask provides a powerful framework allowing parallel processing and automated scaling accessed from a lightweight python notebook. dask-gateway provides a set of services that can be easily installed onto kubernetes using helm. dask-gateway can be installed either as a standalone service or tightly integrated with an instance of JupyterHub.

For standalone installation, follow the instructions at https://gateway.dask.org/install-kube.html#install-dask-gateway, using the default config file at https://github.com/dask/dask-gateway/blob/master/resources/helm/dask-gateway/values.yaml
Dask will automatically try to acquire an external floating ip address for it's load balancer. This can be listed using kubectl, eg::

  kubectl get service --namespace dask-gateway
  NAME                   TYPE           CLUSTER-IP      EXTERNAL-IP       PORT(S)        AGE
  api-dask-gateway       ClusterIP      10.254.61.172   <none>            8000/TCP       3h55m
  traefik-dask-gateway   LoadBalancer   10.254.138.19   128.232.227.222   80:31454/TCP   3h55m

If your firewall does not automatically allow ingress for new floating ip addresses, you will need to contact your cloud support for advice on how to obtain a new address and request ingress. You will then need to reinstall dask specifying your new floating ip address as the address of the load balancer.

Your dask installation can be tested by creating a new Jupyter notebook, then running a simple dask task from within it.
Note that dask is sensitive to mismatches in versions between libraries on the worker images and in the calling client.
The dask_gateway client provides a get_versions method which checks for any potential mismatches.

The following code provides a simple test case for running in a notebook::

  pip install dask_gateway
  from dask_gateway import Gateway
  import dask.array as da
  try:
    gateway = Gateway("address of your dask load balancer external IP")
    cluster = gateway.new_cluster()
    cluster.scale(5)
    client = cluster.get_client()
    client.get_versions(true)
    print('created cluster, allocating random array')
    a = da.random.normal(size=(10000, 10000), chunks=(500, 500))
    print('starting calculation')
    print('mean {0}', a.mean().compute())
    print('shutting down cluster')
    cluster.shutdown()
    print('done')
  except Exception as e:
    print(e)

execute this, and via kubectl you should be able to see dask starting and stopping worker pods on demand, eg::

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
  traefik-dask-gateway-754b78c8-fqcx6                  1/1     Running   0          4h3

Integrated dask and JupyterHub
------------------------------
Although an instance of dask in one namespace can be configured to use authentication from a JupyterHub instance in a separate namespace, if the intention is to provide access to dask purely via JupyterHub, dask and JupyterHub can more easily be installed into the same namespace and integrated using the daskhub charts. This means that dask is only accessible via the JupyterHub instance in the same namespace, and easier to manage.

This will provision a single namespace containing a dask instance, a JupyterHub instance, a shared external load balancer and access to dask only via the JupyterHub instance.
Follow the instructions https://docs.dask.org/en/latest/setup/kubernetes-helm.html#helm-install-dask-for-mulitple-users

This will provision a single namespace containing a dask instance, a JupyterHub instance, a shared external load balancer and access to dask only via the JupyterHub instance.
Important note: as written, the instructions will install into the default namespace. This is very unadvisable! When following the installation instructions, specify a namespace for your installation, eg to install into a dhub namespace::

  helm upgrade --debug --wait --namespace dhub --create-namespace --install --render-subchart-notes  dhub dask/daskhub     --values=secrets.yaml

An example secrets.yaml file::

  jupyterhub:
    proxy:
      secretToken: <token1>
    hub:
      services:
        dask-gateway:
          apiToken: <token2>
      db:
        type: sqlite-memory
    singleuser:
      storage:
        type: sqlite-memory

  dask-gateway:
    gateway:
      auth:
        jupyterhub:
          apiToken: <token2>
          
Following installation, all the services for dask and jupyterhub should be visible in the namespace, including a single load balancer serving both JupyterHub and Dask::

  $ kubectl --namespace=dhub get services
  NAME                        TYPE           CLUSTER-IP       EXTERNAL-IP      PORT(S)        AGE
  api-dhub-dask-gateway       ClusterIP      10.254.196.19    <none>           8000/TCP       46h
  hub                         ClusterIP      10.254.247.105   <none>           8081/TCP       46h
  proxy-api                   ClusterIP      10.254.133.141   <none>           8001/TCP       46h
  proxy-public                LoadBalancer   10.254.172.75    128.232.224.75   80:32574/TCP   46h
  traefik-dhub-dask-gateway   ClusterIP      10.254.122.252   <none>           80/TCP         46h

The code from the previous example can be used with a couple of small changes. 

There is now no need to specify an address, as the notebook will default to using the dask instance in the same namespace, and we use a GatewayCluster object instead of Gateway::

  !pip install dask_gateway
  from dask_gateway import GatewayCluster
  import dask.array as da
  try:
    cluster = GatewayCluster()
    cluster.scale(5)
    client = cluster.get_client()
    client.get_versions(true)
    print('created cluster, allocating random array')
    a = da.random.normal(size=(10000, 10000), chunks=(500, 500))
    print('starting calculation')
    print('mean {0}', a.mean().compute())
    print('shutting down cluster')
    cluster.shutdown()
    print('done')
  except Exception as e:
    print(e)

