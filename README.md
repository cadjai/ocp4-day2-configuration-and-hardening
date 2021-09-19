Role Name
=========

A utility role to OpenShift Container Platform 4.6+ (OCP4) post installation activities.
Some of the roles are to configure a disconnected Operator Lifecycle Manager (OLM), deploying various operators from OLM, add infranodes and relocating OCP4 infrastructure components (ingress controller, image registry, cluster logging, cluster monitoring ...) from worker nodes to infra nodes as well as confiruging tolerations to make sure infra components can come up correctly on infra node. 

Initial infrastructure nodes addition playbook focuses on AWS. 

Requirements
------------
A cluster with a valid registry containing all operators to be installed is configured especially when using this in a disconnected environment. To import opertors refer to https://github.com/cadjai/mirror-ocp4-contents-for-artifactory.git and https://github.com/cadjai/mirror-ocp4-contents-for-artifactory.git for examples of how to import contents to your disconnected cluster. 
A running OCP 4 cluster with valid credentials provided through the variables described below.


Plays and Role Variables
------------------------

- ansible_name_module: The name of the module this role is part of. This is used to track context when this role is run in a playbook as part of other plays.  
- openshift_cli: Openshift client binary used to interact with the cluster api (default to 'oc')
- ocp_cluster_user: The name of the cluster-admin user used to perform the various actions against the cluster.
- ocp_cluster_user_password: The password of the cluster-admin user used to perform the various actions against the cluster.
- ocp_cluster_console_url: The URL of the API the cluster these actions are being applied to.
- ocp_cluster_console_port: The port on which the cluster api is listening (default to 6443)
- staging_dir: The directory where rendered yaml config are placed on the '/tmp'
- disconnected_operatorhub_config: The name of the processed OLM yaml config (default to 'operatorhub-config.yml')
- disconnected_catalog_source_config: The name of the processed catalag source yaml config (default to 'catalog-source-config.yml')
- configure_disconnected_OLM: Indicates wether to apply the disconnected role to the cluster or not. If it was already applied then can be set to false.
- operator_image_content_source_file: The name the Image Content Source config for operators images in the disconnected registry.
- local_repository: The name of repository where the operator images are stored in the disconnected registry (default to 'openshift4/redhat-operators')
- operator_catalogs_to_deploy: The structure containing information about the various operator indices to deploy.These valued as used to generate the catalog source yaml file and the Image Content source file to use for the catalog.
   - catalog_name: The name of the catalog.
   - catalog_index: The name of the operator index in the disconnected registry.
   - catalog_index_tag: The tag of the operator index in the disconnected registry.
   - catalog_publisher: The name of the catalog publisher.
   - content_source_file: The fully qualified location of the image content source policy file for the catalog
   - mirror: Whether the catalog need to be deployed or not.
- operators_to_deploy: The structure containing information about the various operators to deploy.These valued as used to generate the OperatorGroup and operator Subscription objects.
   - catalog_name: The name of the catalog the operator is deployed from.
   - group_name: The name of the operator group object this operator is part of.
   - target_namespace: The namespace being monitored by the operator. This can be set to all or empty for cluster wide operators. 
   - deploy_namespace: The namespace to deploy the operator into.
   - deploy_namespace_description: The description of the namespace to deploy the operator into.
   - deploy:  Whether to deploy the operator or not. Only being used so that the paybook can be reused without affecting previously deployed operators.
   - create_namespace: Whether to create a namespace for the operator or not. Usually the first time this should be set to true if it is a namespaced operator.
   - create_catalog:  Whether to create a dedicated CatalogSource for the operator.
   - create_catalog_group: Whether to create an OperatorGroup config file for the operator.
   - index_image:  The index image to use if a dedicated index is being created for this operator.
   - index_image_tag: The index tag if a dedicated index is being created for this operator.
Note that there are operator sepcific variables not described here (see prometheus, elasticsearch for example of some of those variables).

Variables for infranodes configuration
- infranodes: The structure containing information about the various infrastructure nodes to add and configure. This is configured for AWS but can be adapted for any other cloud provider.
  - aws_region: The AWS region the resource is to be provisioned in.
  - aws_az: The AWS Availability Zone to to provision the node in.
  - role: The Kubernetes role to apply to the node. Default to infra but can also be set to worker or any other string.
  - type: The Kubernetes type to apply to the node. Default to infra but can also be set to worker or any other string.
  - sg: The AWS security group to apply to the node once provisioned. It is assumed that the security group already exists. Default to worker_sg since that is created at cluster deployment.
  - profile: The AWS IAM profile to apply to the node once provisioned. It is assumed that the IAM profile already exists. Default to worker since that is created at cluster deployment.
  - replica_count: The number of node to provision. Default to 1 given that we are provisioning a node per AZ.
  - block_size: The size of the root partition for the RHCOS instance . Default to 120 GB.
  - volume_type: The type of persistence volume to use to support infrastructure resource persistence. Default to gp2.
- relocate_ingress_controller: Indicates whether to move Ingress Controller from currently deployed worker nodes to the new infra nodes. Default to true.
- relocate_image_registry: Indicates whether to move internal Image registry from currently deployed worker nodes to the new infra nodes. Default to true.
- relocate_monitoring: Indicates whether to move cluster monitoring components from currently deployed worker nodes to the new infra nodes. Default to true.
- relocate_logging: Indicates whether to move cluster logging components from currently deployed worker nodes to the new infra nodes. Default to true.

Dependencies
------------
This role uses the https://github.com/cadjai/config-OpenShift-Lifecycle-Manager-Disconnected.git role to configure the disconnected OLM.


Installation and Usage
-----------------------
Clone the repository to where you want to run this from and make sure you can connect to your cluster using the CLI .
Ensure you install all requirements using `ansible-galaxy install -r requirements.yml --force` before performing the next steps.
You will also need to have cluster-admin run in order to run the playbook since the cron job need to run privileged.
Finally before running the playbook make sure to set and update the variables as appropriate to your use case.

Playbooks
---------
To run the main playbook use the ansible-playbook command as follows
`ansible-plabook install-and-configure-operators-from-olm.yml`

To run the infra nodes addition and configuraton  playbook use the ansible-playbook command as follows
`ansible-plabook add-and-configure-infranodes.yml`

License
-------

BSD

Author Information
------------------

An optional section for the role authors to include contact information, or a website (HTML is not allowed).
