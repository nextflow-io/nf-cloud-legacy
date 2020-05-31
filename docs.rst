************
Amazon Cloud
************

Nextflow provides out of the box support for the Amazon AWS cloud allowing you to setup a computing cluster,
deploy it and run your pipeline in the AWS infrastructure in a few commands.

.. warning::
    This feature is deprecated in favour of the :ref:`awscloud-batch` executor.

Configuration
=============

Cloud configuration attributes are provided in the ``nextflow.config`` file as shown in the example below::

    cloud {
        imageId = 'ami-4b7daa32'
        instanceType = 'm4.xlarge'
        subnetId = 'subnet-05222a43'
    }

The above attributes define the virtual machine ID and type to be used and the VPC subnet ID to be applied
in you cluster. Replace these values with the ones of your choice.

Nextflow only requires a Linux image that provides support for `Cloud-init <http://cloudinit.readthedocs.io/>`_
bootstrapping mechanism and includes a Java runtime (version 8) and a Docker engine (version 1.11 or later).

For your convenience the following pre-configured Amazon Linux AMI is available in the *EU Ireland* region:
``ami-4b7daa32``.


AWS credentials
---------------

Nextflow will use the AWS credentials defined in your environment, using the standard AWS variables shown below:

    * ``AWS_ACCESS_KEY_ID``
    * ``AWS_SECRET_ACCESS_KEY``
    * ``AWS_DEFAULT_REGION``

If ``AWS_ACCESS_KEY_ID`` and ``AWS_SECRET_ACCESS_KEY`` are not defined in the environment, Nextflow will attempt to 
retrieve credentials from your ``~/.aws/credentials`` or ``~/.aws/config`` files. The ``default`` profile can be
overridden via the environmental variable ``AWS_PROFILE`` (or ``AWS_DEFAULT_PROFILE``).

Alternatively AWS credentials can be specified in the Nextflow configuration file.
See :ref:`AWS configuration<config-aws>` for more details.

.. note:: Credentials can also be provided by using an IAM Instance Role. The benefit of this approach is that
  it spares you from managing/distributing AWS keys explicitly.
  Read the `IAM Roles <http://docs.aws.amazon.com/AWSEC2/latest/UserGuide/iam-roles-for-amazon-ec2.html>`_ documentation
  and `this blog post <https://aws.amazon.com/blogs/security/granting-permission-to-launch-ec2-instances-with-iam-roles-passrole-permission/>`_ for more details.

User & SSH key
--------------

By default Nextflow creates in each EC2 instance a user with the same name as the one in your local computer and install
the SSH public key available at the path ``$HOME/.ssh/id_rsa.pub``. A different user/key can be specified as shown below::

    cloud {
        userName = 'the-user-name'
        keyFile = '/path/to/ssh/key.pub'
    }

If you want to use a *key-pair* defined in your AWS account and the default user configured in the AMI, specify the
attribute ``keyName`` in place of ``keyFile`` and the name of the existing user specifying the ``userName`` attribute.


Storage
-------

The following storage types can be defined in the cloud instances: *boot*, *instance* and *shared*.

Boot storage
------------

You can set the size of the `root device volume <http://docs.aws.amazon.com/AWSEC2/latest/UserGuide/RootDeviceStorage.html>`_
by specifying the attribute ``bootStorageSize``. For example::

    cloud {
        imageId = 'ami-xxx'
        bootStorageSize = '10 GB'
    }


Instance storage
----------------

Amazon instances can provide one or more `ephemeral storage volumes <http://docs.aws.amazon.com/AWSEC2/latest/UserGuide/InstanceStorage.html>`_,
depending the instance type chosen. This storage can be made available by using the ``instanceStorageMount``
configuration attributes, as shown below::

    cloud {
        imageId = 'ami-xxx'
        instanceStorageMount = '/mnt/scratch'
    }


The mount path can be any of your choice.

.. note:: When the selected instance provides more than one ephemeral storage volume, Nextflow automatically groups all
  of them together in a single logical volume and mounts it to the specified path. Therefore the resulting instance
  storage size is equals to the sum of the sizes of all ephemeral volumes provided by the actual instance
  (this feature requires Nextflow version 0.27.0 or later).

If you want to mount a specific instance storage volume, specify the corresponding device name by using
the ``instanceStorageDevice`` setting in the above configuration. See the Amazon documentation for details on
`EC2 Instance storage <http://docs.aws.amazon.com/AWSEC2/latest/UserGuide/InstanceStorage.html>`_ and
`devices naming <http://docs.aws.amazon.com/AWSEC2/latest/UserGuide/device_naming.html>`_.


Shared file system
------------------

A shared file system can easily be made available in your cloud cluster by using the `Amazon EFS <https://aws.amazon.com/efs/>`_
storage. You will only need to specify in the cloud configuration the EFS file system ID and optionally the
mount path. For example::

    cloud {
        imageId = 'ami-xxx'
        sharedStorageId = 'fs-1803efd1'
        sharedStorageMount = '/mnt/shared'
    }

.. note:: When the attribute ``sharedStorageMount`` is omitted the path ``/mnf/efs`` is used by default.


Cluster deployment
==================

Once defined the configuration settings in the ``nextflow.config`` file you can create the cloud cluster
by using the following command::

    nextflow cloud create my-cluster -c <num-of-nodes>

The string ``my-cluster`` identifies the cluster instance. Replace it with a name of your choice.

Finally replace ``num-of-nodes`` with the actual number of instances that will made-up the cluster.
One node is created as *master*, the remaining as *workers*. If the option ``-c`` is omitted only the *master* node
is created.

.. warning:: You will be charged accordingly the type and the number of instances chosen.


Pipeline execution
==================

Once the cluster initialization is complete, connect to the *master* node using the SSH command which will be displayed by
Nextflow.

.. note:: On MacOS, use the following command to avoid being asked for a pass-phrase even
  you haven't defined one::

    ssh-add -K [private key file]

You can run your Nextflow pipeline as usual, the environment is automatically configured to use the :ref:`Ignite<ignite-page>`
executor. If the Amazon EFS storage is specified in the cloud configuration the Nextflow work directory will
automatically be set in a shared folder in that file system.

The suggested approach is to run your pipeline downloading it from a public repository such
GitHub and to pack the binaries dependencies in a Docker container as described in the
:ref:`Pipeline sharing <sharing-page>` section.

Cluster shutdown
================

When completed shutdown the cluster instances by using the following command::

    nextflow cloud shutdown my-cluster


Cluster auto-scaling
====================

Nextflow integration for AWS cloud provides a native support auto-scaling that allows the computing cluster
to scale-out or scale-down i.e. add or remove computing nodes dynamically at runtime.

This is a critical feature, especially for pipelines crunching not homogeneous dataset, because it allows the
cluster to adapt dynamically to the actual workload computing resources need as they change over the time.

Cluster auto-scaling is enabled by adding the ``autoscale`` option group in the configuration file as shown below::

    cloud {
        imageId = 'ami-xxx'
        autoscale {
            enabled = true
            maxInstances = 10
        }
    }


The above example enables automatic cluster scale-out i.e. new instances are automatically launched and added to the
cluster when tasks remain too long in wait status because there aren't enough computing resources available. The
``maxInstances`` attribute defines the upper limit to which the cluster can grow.

By default unused instances are not removed when are not utilised. If you want to enable automatic cluster scale-down
specify the ``terminateWhenIdle`` attribute in the ``autoscale`` configuration group.

It is also possible to define a different AMI image ID, type and spot price for instances launched by the Nextflow autoscaler.
For example::

    cloud {
        imageId = 'ami-xxx'
        instanceType = 'm4.large'

        autoscale {
            enabled = true
            spotPrice = 0.15
            minInstances = 5
            maxInstances = 10
            imageId = 'ami-yyy'
            instanceType = 'm4.4xlarge'
            terminateWhenIdle = true
        }
    }

By doing that it's is possible to create a cluster with a single node i.e. the master node. Then the autoscaler will
automatically add the missing instances, up to the number defined by the ``minInstances`` attributes. These will have a
different image and type from the master node and will be launched a *spot instances* because the ``spotPrice``
attribute has been specified.


Spot prices
===========

Nextflow includes an handy command to list the current price of EC2 spot instances. Simply type the following
command in your shell terminal::

    nextflow cloud spot-prices

It will print the current spot price for all available instances type, similar to the example below::

    TYPE        PRICE  PRICE/CPU ZONE       DESCRIPTION             CPUS   MEMORY DISK
    t1.micro    0.0044    0.0044 eu-west-1c Linux/UNIX                 1 627.7 MB -
    m4.4xlarge  0.1153    0.0072 eu-west-1a Linux/UNIX (Amazon VPC)   16    64 GB -
    m4.10xlarge 0.2952    0.0074 eu-west-1b Linux/UNIX (Amazon VPC)   40   160 GB -
    m4.large    0.0155    0.0077 eu-west-1b Linux/UNIX (Amazon VPC)    2     8 GB -
    m4.2xlarge  0.0612    0.0077 eu-west-1a Linux/UNIX (Amazon VPC)    8    32 GB -
    m4.xlarge   0.0312    0.0078 eu-west-1a Linux/UNIX (Amazon VPC)    4    16 GB -
    c4.8xlarge  0.3406    0.0095 eu-west-1c Linux/UNIX (Amazon VPC)   36    60 GB -
    m1.xlarge   0.0402    0.0100 eu-west-1b Linux/UNIX                 4    15 GB 4 x 420 GB
    c4.4xlarge  0.1652    0.0103 eu-west-1b Linux/UNIX (Amazon VPC)   16    30 GB -
    c1.xlarge   0.0825    0.0103 eu-west-1a Linux/UNIX                 8     7 GB 4 x 420 GB
    m1.medium   0.0104    0.0104 eu-west-1b Linux/UNIX (Amazon VPC)    1   3.8 GB 1 x 410 GB
    c3.8xlarge  0.3370    0.0105 eu-west-1a Linux/UNIX                32    60 GB 2 x 320 GB
    c3.2xlarge  0.0860    0.0108 eu-west-1c Linux/UNIX                 8    15 GB 2 x 80 GB
    c3.4xlarge  0.1751    0.0109 eu-west-1c Linux/UNIX (Amazon VPC)   16    30 GB 2 x 160 GB
    m3.2xlarge  0.0869    0.0109 eu-west-1c Linux/UNIX (Amazon VPC)    8    30 GB 2 x 80 GB
    r3.large    0.0218    0.0109 eu-west-1c Linux/UNIX                 2  15.2 GB 1 x 32 GB
    :


It's even possible to refine the showed data by specifying a filtering and ordering criteria. For example::

    nextflow cloud spot-prices -sort pricecpu -filter "cpus==4"


It will only print instance types having 4 cpus and sorting them by the best price per cpu.


Advanced configuration
======================


The ``cloud`` scope allows you to define the settings of the computing cluster that can be deployed in the cloud
by Nextflow.

The following settings are available:

=========================== ================
Name                        Description
=========================== ================
bootStorageSize             Boot storage volume size e.g. ``10 GB``.
imageId                     Identifier of the virtual machine(s) to launch e.g. ``ami-43f49030``.
instanceRole                IAM role granting required permissions and authorizations in the launched instances.
                            When specifying an IAM role no access/security keys are installed in the cluster deployed in the cloud.
instanceType                Type of the virtual machine(s) to launch e.g. ``m4.xlarge``.
instanceStorageMount        Ephemeral instance storage mount path e.g. ``/mnt/scratch``.
instanceStorageDevice       Ephemeral instance storage device name e.g. ``/dev/xvdc`` (optional).
keyName                     SSH access key name given by the cloud provider.
keyHash                     SSH access public key hash string.
keyFile                     SSH access public key file path.
securityGroup               Identifier of the security group to be applied e.g. ``sg-df72b9ba``.
sharedStorageId             Identifier of the shared file system instance e.g. ``fs-1803efd1``.
sharedStorageMount          Mount path of the shared file system e.g. ``/mnt/efs``.
subnetId                    Identifier of the VPC subnet to be applied e.g. ``subnet-05222a43``.
spotPrice                   Price bid for spot/preemptive instances.
userName                    SSH access user name (don't specify it to use the image default user name).
autoscale                   See below.
=========================== ================

The autoscale configuration group provides the following settings:

=========================== ================
Name                        Description
=========================== ================
enabled                     Enable cluster auto-scaling.
terminateWhenIdle           Enable cluster automatic scale-down i.e. instance terminations when idle (default: ``false``).
idleTimeout                 Amount of time in idle state after which an instance is candidate to be terminated (default: ``5 min``).
starvingTimeout             Amount of time after which one ore more tasks pending for execution trigger an auto-scale request (default: ``5 min``).
minInstances                Minimum number of instances in the cluster.
maxInstances                Maximum number of instances in the cluster.
imageId                     Identifier of the virtual machine(s) to launch when new instances are added to the cluster.
instanceType                Type of the virtual machine(s) to launch when new instances are added to the cluster.
spotPrice                   Price bid for spot/preemptive instances launched while auto-scaling the cluster.
=========================== ================


************
Google Cloud
************

Nextflow provides out-of-the-box support for the `Google Cloud Platform <https://cloud.google.com/>`_
enabling the seamless deployment and execution of Nextflow pipelines over Google cloud services.

The execution can be done either deploying a Nextflow managed cluster using `Google Compute Engine <https://cloud.google.com/compute/>`_
instances or via the `Genomics Pipelines <https://cloud.google.com/genomics/>`_ managed service.

.. note::
    This feature requires Nextflow version 19.01.0 or later.

.. warning::
    This feature is deprecated in favour of :ref:`google-lifesciences` executor.

Requirements
============

Nextflow
--------
The support for Google Cloud requires Nextflow version ``19.01.0``. To install it define the following variables
in your system environment::

    export NXF_VER=19.01.0
    export NXF_MODE=google

Then run the following command::

    curl https://get.nextflow.io | bash

Complete the installation copying the ``nextflow`` launcher script in a directory in your system ``PATH`` (optional).


Credentials
-----------

To allow the deployment in the Google Cloud you need to configure the security credentials using
a *Security account key* JSON file.

Nextflow looks for this file using the ``GOOGLE_APPLICATION_CREDENTIALS`` variable that
has to be defined in the launching environment.

If you don't have it, download the credentials file from the Google Cloud Console following these steps:

* Open the `Google Cloud Console <https://console.cloud.google.com>`_
* Go to APIs & Services → Credentials
* Click on the *Create credentials* (blue) drop-down and choose *Service account key*, in the following page
* Select an existing *Service account* or create a new one if needed
* Select JSON as *Key type*
* Click the *Create* button and download the JSON file giving a name of your choice e.g. ``creds.json``.

Finally define the following variable replacing the path in the example with the one of your
credentials file just downloaded::

    export GOOGLE_APPLICATION_CREDENTIALS=/path/your/file/creds.json


Compute Engine
==============

When using this feature Nextflow allows you set up an ephemeral computing cluster in the Google Cloud platform
and use it to deploy your pipeline execution.

Configuration
-------------

Cloud configuration attributes are provided in the ``nextflow.config`` file as shown in the example below::

    cloud {
        driver = 'google'
        imageId = 'rare-lattice-222412/global/images/seqvm-1'
        instanceType = 'n1-highcpu-8'
    }

    google {
        project = 'your-project-id'
        zone = 'us-central1-f'
    }

The above attributes define the image ID and instance type to be used along with the project and zone to be used.
Replace these values with the ones of your choice.

.. tip:: Make sure to specify the ``google`` as cloud driver value as shown in the above example.

Nextflow requires a Linux image that provides support for `Cloud-init <http://cloudinit.readthedocs.io/>`_
bootstrapping mechanism and includes the Java runtime (version 8 or later) and the Docker engine (version 1.11 or later).

The very first time you will need to create a custom machine image containing these two software packages and use it
to deploy your pipeline execution.

Refer to the Google cloud documentation to learn `how to create a custom image <https://cloud.google.com/compute/docs/images/create-delete-deprecate-private-images>`_.

User & SSH key management
-------------------------

By default Nextflow creates in each GCE instance a user with the same name as the one in your local computer and install
the SSH public key available at the path ``$HOME/.ssh/id_rsa.pub``.

A different user/key can be specified as shown below::

    cloud {
        driver = 'google'
        userName = 'the-user-name'
        keyFile = '/path/to/ssh/key.pub'
    }

Storage
-------

Both input data and the pipeline work directory must be located in one or more `Google Storage <https://cloud.google.com/storage/>`_ buckets.
Make sure your security credentials allows you to access these buckets.

Cluster deployment
------------------

Once you have defined the configuration settings in the ``nextflow.config`` file you can create the cloud cluster by
using the following command::

    nextflow cloud create my-cluster -c <num-of-nodes>

The string ``my-cluster`` identifies the cluster instance. Replace it with a name of your choice.

Finally replace ``<num-of-nodes>`` with the actual number of instances that will make-up the cluster. One node is
created as master, the remaining as workers. If the option ``-c`` is omitted only the **master** node is created.

.. warning:: You will be charged accordingly the type and the number of instances chosen.

The console will display the configuration that you have defined and ask you to confirm creation of the cluster with the
configuration displayed. It will take some time for the cluster to deploy. You should be able to track the status of the
deployed in the administration console for the Google Cloud Platform under VM instances in the Compute Engine section.


Pipeline execution
------------------

Once the master node is available, Nextflow will display the SSH command to connect to the master node. Use
that command to connect to the cluster.

.. note:: On MacOS, use the following command to avoid being asked for a pass-phrase even
  you haven't defined one::

    ssh-add -K [private key file]

The suggested approach is to run your pipeline downloading it from a public repository such as GitHub and to pack the
binaries dependencies in a Docker container as described in the :ref:`Pipeline sharing <sharing-page>` section.

.. warning:: Before running any Nextflow command, make sure the file ``READY`` has been created in the home directory.
  If you can't find it, it means that the initialisation process is still on-going. Wait a few seconds until it completes.

Then, you can run Nextflow as usual. For example::

    ./nextflow run rnaseq-nf -profile gcp -work-dir gs://my-bucket/work


.. tip:: Make sure to specify a Google Storage path, containing at a bucket sub-directory, as the Nextflow work directory
  and as a location for pipeline input data.

.. note:: The ``nextflow`` launcher script is created in the instance ``HOME`` directory.

Cluster shutdown
----------------

When completed, shutdown the cluster instances by using the following command::

    nextflow cloud shutdown my-cluster

Replace ``my-cluster`` with the name used in your execution.

Preemptible instances
---------------------

An optional parameter allows you to set the instance to be preemptible. Both master and worker instances can be set to
be preemptible. The following example shows a cluster configuration with a preemptible setting::

    cloud {
        imageId = 'rare-lattice-222412/global/images/seqvm-1'
        instanceType = 'n1-highcpu-8'
        preemptible = true
    }

Setting an instance to preemptible allows the administrator to kill the VM at will and may affect the pricing of the
instance.

Cluster auto-scaling
--------------------

Nextflow integration for Google Cloud Engine provides a native support auto-scaling that allows the computing cluster
to scale up or scale down i.e., add or remove computing nodes dynamically at runtime.

This is a critical feature, especially for pipelines crunching non-homogeneous datasets, because it allows the cluster
to adapt dynamically to the actual workload computing resources needed as they change over time.

Cluster auto-scaling is enabled by adding the autoscale option group in the configuration file as shown below::

    cloud {
        imageId = 'rare-lattice-222412/global/images/seqvm-1'
        autoscale {
            enabled = true
            maxInstances = 10
        }
    }


The above example enables automatic cluster scale-out i.e. new instances are automatically launched and added to the
cluster when tasks remain too long in wait status because there aren't enough computing resources available. The
``maxInstances`` attribute defines the upper limit to which the cluster can grow.

By default unused instances are not removed when they are not utilised. If you want to enable automatic cluster scale-down
specify the ``terminateWhenIdle`` attribute in the ``autoscale`` configuration group.

It is also possible to define a different machine image IDs, type and spot price for instances launched by the Nextflow
autoscaler. For example::

    cloud {
        imageId = 'your-project/global/images/xxx'
        instanceType = 'n1-highcpu-8'
        preemptible = false
        autoscale {
            enable = true
            preemptible = true
            minInstances = 5
            maxInstances = 10
            imageId = 'your-project/global/images/yyy'
            instanceType = 'n1-highcpu-8'
            terminateWhenIdle = true
        }
    }

By doing so it is possible to create a cluster with a single node i.e. the master node. The autoscaler will then
automatically add the missing instances, up to the number defined by the ``minInstances`` attributes.


Limitation
----------

* Pipeline input data must and work directory must be located on `Google Storage <https://cloud.google.com/storage/>`_.
  Other local or remote data sources are not supported at this time.

* The compute nodes local storage is the default assigned by the Compute Engine service for the chosen machine (instance) type.
  Currently it is not possible to specify a custom disk size for local storage.


Advanced configuration
----------------------

Read :ref:`Cloud configuration<config-cloud>` section to learn more about advanced cloud configuration options.


.. _google-pipelines:

Genomics Pipelines
==================

.. warning::
  The support for Google Pipelines API is deprecated. The `google-lifesciences`_ instead.

`Genomics Pipelines <https://cloud.google.com/genomics/>`_ is a managed computing service that allows the execution of
containerized workloads in the Google Cloud Platform infrastructure.

Nextflow provides built-in support for Genomics Pipelines API which allows the seamless deployment of a Nextflow pipeline
in the cloud, offloading the process executions through the Pipelines service.

.. warning:: This API works well for coarse-grained workloads i.e. long running jobs. It's not suggested the use
  this feature for pipelines spawning many short lived tasks.

.. _google-pipelines-config:

Configuration
-------------

Make sure to have defined in your environment the ``GOOGLE_APPLICATION_CREDENTIALS`` variable.
See the section `Requirements`_ for details.

.. tip:: Make sure to have enabled Genomics API to use this feature. To learn how to enable it
  follow `this link <https://cloud.google.com/genomics/docs/quickstart>`_.

Create a ``nextflow.config`` file in the project root directory. The config must specify the following parameters:

* Google Pipelines as Nextflow executor i.e. ``process.executor = 'google-pipelines'``.
* The Docker container images to be used to run pipeline tasks e.g. ``process.container = 'biocontainers/salmon:0.8.2--1'``.
* The Google Cloud `project` ID to run in e.g. ``google.project = 'rare-lattice-222412'``.
* The Google Cloud `region` or `zone`. You need to specify either one, **not** both. Multiple regions or zones can be
  specified by separating them with a comma e.g. ``google.zone = 'us-central1-f,us-central-1-b'``.

Example::

    process {
        executor = 'google-pipelines'
        container = 'your/container:latest'
    }

    google {
        project = 'your-project-id'
        zone = 'europe-west1-b'
    }


.. warning:: Make sure to specify in the above setting the project ID not the project name.

.. Note:: A container image must be specified to deploy the process execution. You can use a different Docker image for
  each process using one or more :ref:`config-process-selectors`.

Process definition
------------------
Processes can be defined as usual and by default the ``cpus`` and ``memory`` directives are used to instantiate a custom
machine type with the specified compute resources.  If ``memory`` is not specified, 1GB of memory is allocated per cpu.
A persistent disk will be created with size corresponding to the ``disk`` directive.  If ``disk`` is not specified, the
instance default is chosen to ensure reasonable I/O performance.

The process ``machineType`` directive may optionally be used to specify a predefined Google Compute Platform `machine type <https://cloud.google.com/compute/docs/machine-types>`_
If specified, this value overrides the ``cpus`` and ``memory`` directives.
If the ``cpus`` and ``memory`` directives are used, the values must comply with the allowed custom machine type `specifications <https://cloud.google.com/compute/docs/instances/creating-instance-with-custom-machine-type#specifications>`_ .  Extended memory is not directly supported, however high memory or cpu predefined
instances may be utilized using the ``machineType`` directive

Examples::

    process custom_resources_task {
        cpus 8
        memory '40 GB'
        disk '200 GB'

        """
        <Your script here>
        """
    }

    process predefined_resources_task {
        machineType 'n1-highmem-8'

        """
        <Your script here>
        """
    }

.. note:: This feature requires Nextflow 19.07.0 or later.

Pipeline execution
------------------

The pipeline can be launched either in a local computer or a cloud instance. Pipeline input data can be stored either
locally or in a Google Storage bucket.

The pipeline execution must specify a Google Storage bucket where the workflow's intermediate results are stored using
the ``-work-dir`` command line options. For example::

    nextflow run <script or project name> -work-dir gs://my-bucket/some/path


.. tip:: Any input data **not** stored in a Google Storage bucket will automatically be transferred to the
  pipeline work bucket. Use this feature with caution being careful to avoid unnecessary data transfers.

Hybrid execution
----------------

Nextflow allows the use of multiple executors in the same workflow application. This feature enables the deployment
of hybrid workloads in which some jobs are executed in the local computer or local computing cluster and
some other jobs are offloaded to Google Pipelines service.

To enable this feature use one or more :ref:`config-process-selectors` in your Nextflow configuration file to apply
the Google Pipelines *executor* only to a subset of processes in your workflow.
For example::


    process {
        withLabel: bigTask {
            executor = 'google-pipelines'
            container = 'my/image:tag'
        }
    }

    google {
        project = 'your-project-id'
        zone = 'europe-west1-b'
    }


Then deploy the workflow execution using the ``-bucket-dir`` to specify a Google Storage path
for the jobs computed by the Google Pipeline service and, optionally, the ``-work-dir`` to
specify the local storage for the jobs computed locally::

    nextflow run <script or project name> -bucket-dir gs://my-bucket/some/path

.. warning:: The Google Storage path needs to contain at least sub-directory. Don't use only the
  bucket name e.g. ``gs://my-bucket``.

Limitation
----------

* Currently it's not possible to specify a disk type different from the default one assigned
  by the service depending the chosen instance type.


Troubleshooting
---------------

* Make sure to have enabled Compute Engine API, Genomics API and Cloud Storage Service in the
  `APIs & Services Dashboard <https://console.cloud.google.com/apis/dashboard>`_ page.

* Make sure to have enough compute resources to run your pipeline in your project
  `Quotas <https://console.cloud.google.com/iam-admin/quotas>`_ (i.e. Compute Engine CPUs,
  Compute Engine Persistent Disk, Compute Engine In-use IP addresses, etc).

* Make sure your security credentials allows you to access any Google Storage bucket
  where input data and temporary files are stored.

Google Pipelines debugging information can be enabled using the ``-trace`` command line option
as shown below::

    nextflow -trace nextflow.cloud.google.pipelines run <your_project_or_script_name>

    