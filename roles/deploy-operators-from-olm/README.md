Role Name
=========

This role is used to deploy openshift operators from the configure Operator Lifecycle Manager using the CLI to deploy said operators to help with automation.

Requirements
------------

Any pre-requisites that may not be covered by Ansible itself or the role should be mentioned here. For instance, if the role uses the EC2 module, it may be a good idea to mention in this section that the boto package is required.

Role Variables
--------------

- ansible_name_module: The name of the module this role is part of. This is used to track context when this role is run in a playbook as part of other plays.
- openshift_cli: Openshift client binary used to interact with the cluster api (default to 'oc')
- staging_dir: The directory where rendered yaml config are placed on the '/tmp'
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



Installation and Usage
-----------------------
Clone the repository to where you want to run this from and make sure you can connect to your cluster using the CLI .
You will also need to have cluster-admin run in order to run the playbook since the cron job need to run privileged.
Finally before running the playbook make sure to set and update the variables as appropriate to your use case.


Dependencies
------------

A list of other roles hosted on Galaxy should go here, plus any details in regards to parameters that may need to be set for other roles, or variables that are used from other roles.

Example Playbook
----------------

To deploy operators using a playbook that import the role use a playbook similar to the following:

    - hosts: localhost
      vars:
        ansible_python_interpreter: /usr/bin/python3
        module: "install-operators-from-OLM"
        ansible_name_module: " Deploy Operator from OLM | {{ module }}"
      vars_files:
        - 'vars/vault.yml'
        - 'vars/operators.yml' 
      roles:
         - { role: deploy-operators-from-olm }

The vault.yml will contains your cluster authentication infromation 
The operators.yml will contain the operators to deploy. 

To deploy a single operator with inline variable use the following sample

    - hosts: localhost
      vars:
        ansible_python_interpreter: /usr/bin/python3
        module: "install-operators-from-OLM"
        ansible_name_module: " Deploy Operator from OLM | {{ module }}"
        staging_dir: '/tmp'
        ocp_cluster_admin_user: ''
        ocp_cluster_admin_password: ''
        ocp_cluster_console_url: ''
        ocp_cluster_console_port: ''
        operators_to_deploy:
          gitlab-runner-operator:
            group_name: 'gitlab-runner-operator-group'
            catalog_name: 'certified-operators'
            sub_channel: 'beta'
            deploy_namespace: 'gitlab-express-dev'  ## used for where the operator is deployed
            target_namespace: 'gitlab-express-dev'  ## used for the scope of operator
            deploy_namespace_description: 'gitlab-express-dev'
            deploy: 'true'
            create_namespace: 'true'
            create_catalog: 'false'
            create_catalog_group: 'true'
            index_image: 'certified-operator-index'

      tasks:
        - name: Install required pip library
          pip:
            name: openshift
            state: present

        - name: Ensure Proper Python dependency is installed for Openshift
          python_requirements_facts:
            dependencies:
              - openshift
              - requests

        - name: Authenticate with the API
          command: >
            {{ openshift_cli }} login \
              --token {{ ocp_cluster_token }} \
              -p {{ ocp_cluster_user_password }} \
              --insecure-skip-tls-verify=true {{ ocp_cluster_console_url }}:{{ ocp_cluster_console_port | d('6443', true) }}
          when:
            - ocp_cluster_token is defined and ocp_cluster_token != ""
          register: login_out

        - name: Authenticate with the API
          command: >
            {{ openshift_cli }} login \
              -u {{ ocp_cluster_user }} \
              -p {{ ocp_cluster_user_password }} \
              --insecure-skip-tls-verify=true {{ ocp_cluster_console_url }}:{{ ocp_cluster_console_port | d('6443', true) }}
          when:
            - ocp_cluster_token is undefined or ocp_cluster_token == ""
            - ocp_cluster_user is defined and ocp_cluster_user != ""
            - ocp_cluster_user_password is defined and ocp_cluster_user_password != ""
          register: login_out

        - name: Deploy operators from OLM
          import_role:
            name: deploy-operators-from-olm

License
-------

BSD

Author Information
------------------

An optional section for the role authors to include contact information, or a website (HTML is not allowed).
