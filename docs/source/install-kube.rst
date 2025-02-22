Install on a Kubernetes Cluster
===============================

Here we provide instructions for installing and configuring
``dask-gateway-server`` on a `Kubernetes Cluster`_.


Create a Kubernetes Cluster (optional)
--------------------------------------

If you don't already have a cluster running, you'll want to create one. There
are plenty of guides online for how to do this. We recommend following the
excellent `documentation provided by zero-to-jupyterhub-k8s`_.


Install Helm (optional)
-----------------------

If you don't already have Helm_ installed, you'll need to install it locally,
and ensure ``tiller`` is running on your cluster. As with above, there are
plenty of instructional materials online for doing this. We recommend following
the `guide provided by zero-to-jupyterhub-k8s`_.


Install Dask-Gateway
--------------------

At this point you should have a Kubernetes cluster with Helm installed and
configured. You are now ready to install Dask-Gateway on your cluster.


Add the Helm Chart Repository
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

To avoid downloading the chart locally from GitHub, you can use the
`Dask-Gateway Helm chart repository`_.

.. code-block:: shell

    $ helm repo add dask-gateway https://dask.org/dask-gateway-helm-repo/
    $ helm repo update


Configuration
~~~~~~~~~~~~~

The Helm chart provides access to configure most aspects of the
``dask-gateway-server``. These are provided via a configuration YAML_ file (the
name of this file doesn't matter, we'll use ``config.yaml``).

At a minimum, you'll need to set a value for ``gateway.proxyToken``. This is a
random hex string representing 32 bytes, used as a security token between the
gateway and its proxies. You can generate this using ``openssl``:

.. code-block:: shell

    $ openssl rand -hex 32

Write the following into a new file ``config.yaml``, replacing ``<RANDOM
TOKEN>`` with the output of the previous command above.

.. code-block:: yaml

    gateway:
      proxyToken: "<RANDOM TOKEN>"

The Helm chart exposes many more configuration values, see the `default
values.yaml file`_ for more information.


Install the Helm Chart
~~~~~~~~~~~~~~~~~~~~~~

To install the Dask-Gateway Helm chart, run the following command:

.. code-block:: shell

    RELEASE=dask-gateway
    NAMESPACE=dask-gateway
    VERSION=0.6.0

    helm upgrade --install \
        --namespace $NAMESPACE \
        --version $VERSION \
        --values path/to/your/config.yaml \
        $RELEASE \
        dask-gateway/dask-gateway

where:

- ``RELEASE`` is the `Helm release name`_ to use (we suggest ``dask-gateway``,
  but any release name is fine).
- ``NAMESPACE`` is the `Kubernetes namespace`_ to install the gateway into (we
  suggest ``dask-gateway``, but any namespace is fine).
- ``VERSION`` is the Helm chart version to use. To use the latest published
  version you can omit the ``--version`` flag entirely. See the `Helm chart
  repository`_ for an index of all available versions.
- ``path/to/your/config.yaml`` is the path to your ``config.yaml`` file created
  above.

Running this command may take some time, as resources are created and images
are downloaded. When everything is ready, running the following command will
show the ``EXTERNAL-IP`` addresses for all ``LoadBalancer`` services (highlighted
below).

.. code-block:: shell
    :emphasize-lines: 4,6

    $ kubectl get service --namespace dask-gateway
    NAME                            TYPE           CLUSTER-IP      EXTERNAL-IP      PORT(S)          AGE
    scheduler-api-dask-gateway      ClusterIP      10.51.245.233   <none>           8001/TCP         6m54s
    scheduler-public-dask-gateway   LoadBalancer   10.51.253.105   35.202.68.87     8786:31172/TCP   6m54s
    web-api-dask-gateway            ClusterIP      10.51.250.11    <none>           8001/TCP         6m54s
    web-public-dask-gateway         LoadBalancer   10.51.247.160   146.148.58.187   80:30304/TCP     6m54s

At this point, you have a fully running ``dask-gateway-server``.


Connecting to the gateway
-------------------------

To connect to the running ``dask-gateway-server``, you'll need the external
IPs from both the ``web-public-*`` and ``scheduler-public-*`` services above.
The ``web-public-*`` service provides access to API requests, and also proxies
out the `Dask Dashboards`_. The ``scheduler-public-*`` service proxies TCP
traffic between Dask clients and schedulers.

To connect, create a :class:`dask_gateway.Gateway` object, specifying the both
addresses (the ``scheduler-proxy-*`` IP/port goes under ``proxy_address``).
Using the same values as above:

.. code-block:: python

    >>> from dask_gateway import Gateway
    >>> gateway = Gateway(
    ...     "http://146.148.58.187",
    ...     proxy_address="tls://35.202.68.87:8786"
    ... )

You should now be able to use the gateway client to make API calls. To verify
this, call :meth:`dask_gateway.Gateway.list_clusters`. This should return an
empty list as you have no clusters running yet.

.. code-block:: python

    >>> gateway.list_clusters()
    []


Shutting everything down
------------------------

When you're done with the gateway, you'll want to delete your deployment and
clean everything up. You can do this with ``helm delete``:

.. code-block:: shell

    $ helm delete --purge $RELEASE


Additional configuration
------------------------

Here we provide a few configuration snippets for common deployment scenarios.
For all available configuration fields see the :ref:`helm-chart-reference`.


Using ``extraPodConfig``/``extraContainerConfig``
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

The `Kubernetes API`_ is large, and not all configuration fields you may want
to set on scheduler/worker pods are directly exposed by the Helm chart. To
address this, we provide a few fields for forwarding configuration directly to
the underlying kubernetes objects:

- ``gateway.clusterManager.scheduler.extraPodConfig``
- ``gateway.clusterManager.scheduler.extraContainerConfig``
- ``gateway.clusterManager.worker.extraPodConfig``
- ``gateway.clusterManager.worker.extraContainerConfig``

These allow configuring any unexposed fields on the pod/container for
schedulers and workers respectively. Each takes a mapping of key-value pairs,
which is deep-merged with any settings set by dask-gateway itself (with
preference given to the ``extra*Config`` values). Note that keys should be
``camelCase`` (rather than ``snake_case``) to match those in the kubernetes
API.

For example, this can be useful for setting things like tolerations_ or `node
affinities`_ on scheduler or worker pods. Here we configure a node
anti-affinity for scheduler pods to avoid `preemptible nodes`_:

.. code-block:: yaml

  gateway:
    clusterManager:
      scheduler:
        extraPodConfig:
          affinity:
            nodeAffinity:
              requiredDuringSchedulingIgnoredDuringExecution:
                nodeSelectorTerms:
                  - matchExpressions:
                    - key: cloud.google.com/gke-preemptible
                      operator: DoesNotExist

For information on allowed fields, see the Kubernetes documentation:

- `PodSpec Configuration <https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.15/#podspec-v1-core>`__
- `Container Configuration <https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.15/#container-v1-core>`__


Using ``extraConfig``
~~~~~~~~~~~~~~~~~~~~~

Not all configuration options have been exposed via the helm chart. To set
unexposed options, you can use the ``gateway.extraConfig`` field. This takes
either:

- A python code-block (as a string) to append to the end of the generated
  ``dask_gateway_config.py`` file.
- A map of keys -> code-blocks. When applied in this form, code-blocks are
  appended in alphabetical order by key (the keys themselves are meaningless).
  This allows merging multiple ``values.yaml`` files together, as Helm can
  natively merge maps.

For example, here we use ``gateway.extraConfig`` to set
:data:`c.DaskGateway.cluster_manager_options`, exposing options for worker
resources and image (see :doc:`cluster-options` for more information).

.. code-block:: yaml

    gateway:
      extraConfig: |
        from dask_gateway_server.options import Options, Integer, Float, String

        def option_handler(options):
            return {
                "worker_cores": options.worker_cores,
                "worker_memory": "%fG" % options.worker_memory,
                "image": options.image,
            }

        c.DaskGateway.cluster_manager_options = Options(
            Integer("worker_cores", 2, min=1, max=4, label="Worker Cores"),
            Float("worker_memory", 4, min=1, max=8, label="Worker Memory (GiB)"),
            String("image", default="daskgateway/dask-gateway:latest", label="Image"),
            handler=option_handler,
        )

For information on all available configuration options, see the
:doc:`api-server` (in particular, the :ref:`kube-cluster-manager-config`
section).


Authenticating with JupyterHub
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

JupyterHub_ provides a multi-user interactive notebook_ environment. Through
the zero-to-jupyterhub-k8s_ project, many companies and institutions have setup
JuypterHub to run on Kubernetes. When deploying Dask-Gateway alongside
JupyterHub, you can configure Dask-Gateway to use JupyterHub for
authentication. To do this, we register ``dask-gateway`` as a `JupyterHub
Service`_.

First we need to generate an API Token - this is commonly done using
``openssl``:

.. code-block:: shell

    $ openssl rand -hex 32

Then add the following lines to your ``config.yaml`` file:

.. code-block:: yaml

    gateway:
      auth:
        type: jupyterhub
        jupyterhub:
          apiToken: "<API TOKEN>"

replacing ``<API TOKEN>`` with the output from above.

If you're not deploying Dask-Gateway in the same cluster and namespace as
JupyterHub, you'll also need to specify JupyterHub's API url. This is usually
of the form ``https://<JUPYTERHUB-HOST>:<JUPYTERHUB-PORT>/hub/api``. If
JupyterHub and Dask-Gateway are on the same cluster and namespace you can omit
this configuration key, the address will be inferred automatically.

.. code-block:: yaml

    gateway:
      auth:
        type: jupyterhub
        jupyterhub:
          apiToken: "<API TOKEN>"
          apiUrl: "<API URL>"

You'll also need to add the following to the ``config.yaml`` file for your
JupyterHub Helm Chart.

.. code-block:: yaml

    hub:
      services:
        dask-gateway:
          apiToken: "<API TOKEN>"

again, replacing ``<API TOKEN>`` with the output from above.

With this configuration, JupyterHub will be used to authenticate requests
between users and the ``dask-gateway-server``. Note that users will need to add
``auth="jupyterhub"`` when they create a Gateway :class:`dask_gateway.Gateway`
object.

.. code-block:: python

    >>> from dask_gateway import Gateway
    >>> gateway = Gateway(
    ...     "http://146.148.58.187",
    ...     proxy_address="tls://35.202.68.87:8786",
    ...     auth="jupyterhub",
    ... )


.. _helm-chart-reference:

Helm chart reference
--------------------

The full `default values.yaml file`_ for the dask-gateway Helm chart is included
here for reference:

.. literalinclude:: ../../resources/helm/dask-gateway/values.yaml
    :language: yaml


.. _Kubernetes Cluster: https://kubernetes.io/
.. _Helm: https://helm.sh/
.. _documentation provided by zero-to-jupyterhub-k8s: https://zero-to-jupyterhub.readthedocs.io/en/latest/create-k8s-cluster.html
.. _zero-to-jupyterhub-k8s: https://zero-to-jupyterhub.readthedocs.io/en/latest/
.. _guide provided by zero-to-jupyterhub-k8s: https://zero-to-jupyterhub.readthedocs.io/en/latest/setup-helm.html
.. _Helm chart repository:
.. _dask-gateway helm chart repository: https://dask.org/dask-gateway-helm-repo/
.. _dask-gateway github repo: https://github.com/dask/dask-gateway/
.. _resources/helm subdirectory: https://github.com/dask/dask-gateway/tree/master/resources/helm
.. _default values.yaml file: https://github.com/dask/dask-gateway/blob/master/resources/helm/dask-gateway/values.yaml
.. _Helm release name: https://docs.helm.sh/glossary/#release
.. _Kubernetes namespace: https://kubernetes.io/docs/concepts/overview/working-with-objects/namespaces/
.. _Dask Dashboards: https://docs.dask.org/en/latest/diagnostics-distributed.html
.. _yaml: https://en.wikipedia.org/wiki/YAML
.. _JupyterHub: https://jupyterhub.readthedocs.io/
.. _notebook: https://jupyter.org/
.. _JupyterHub Service: https://jupyterhub.readthedocs.io/en/stable/getting-started/services-basics.html
.. _Kubernetes API: https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.15/
.. _tolerations: https://kubernetes.io/docs/concepts/configuration/taint-and-toleration/
.. _node affinities: https://kubernetes.io/docs/concepts/configuration/assign-pod-node/
.. _preemptible nodes: https://cloud.google.com/blog/products/containers-kubernetes/cutting-costs-with-google-kubernetes-engine-using-the-cluster-autoscaler-and-preemptible-vms
