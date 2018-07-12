===========================
Using openstack-keypair-apb
===========================

So far setup of the development environment is not trivial and there are open
issues:

- https://github.com/openshift/ansible-service-broker/issues/1006
- https://github.com/openshift/ansible-service-broker/issues/1007
- https://github.com/automationbroker/automation-broker-apb/pull/27


Dev Env preparation
-------------------

- check out `https://github.com/fusor/catasb`
- create catasb/config/my_vars.yaml with following content:

    ---
    dockerhub_org: ansibleplaybookbundle

- invoke `$ catasb/local/linux/run_setup_local.sh`
- take a coffee
- invoke `$ oc annotate route -n openshift-automation-service-broker asb --overwrite haproxy.router.openshift.io/timeout=300s`
  (https://github.com/automationbroker/automation-broker-apb/pull/27)
- login to UI and follow the link `https://172.17.0.1:8443/console/project/openshift-automation-service-broker/browse/config-maps/broker-config`
- edit the `broker-config` configMap and set `broker.launch_apb_on_bind: true`
  (https://blog.openshift.com/asynchronous-bind-with-the-automation-broker/)
- trigger `openshift-automation-service-broker` deployment (in the same project)
- install APB tool `https://github.com/ansibleplaybookbundle/ansible-playbook-bundle`


APB test
--------

- from our APB dir (here `openstack-service-broker/apb/openstack-keypair-apb`)
  invoke `$ apb push` (this builds and pushes Docker image into the local
  registry)
- in the Openshift UI switch to suitable project (i.e. `myproject`)
- go to the catalog and install `Openstack KeyPair (APB)`. Populate cloud
  connection configuration
- switch to `Provisioned Services` and find new Service
- after it becomes `available` choose `bind` and fill cloud_name and key_name
- the binding should shortly become available and a secret with KeyPair info
  generated

Notes
-----

- Due to the https://github.com/openshift/ansible-service-broker/issues/1007
  it is currently not possible to remove bindings when async binding is enabled
- Openstack modules are currently not able to accept `os_client_config` as
  strings, therefore it the playbook those are temporarily flushed to disk.
  This can be avoided by manual parsing of the yaml and filling the supported
  `auth` parameter of the modules. Another alternative is to create and pack
  ansible module, which will access openstacksdk directly. This way we would
  not depend on Ansible modules and have a better control. It is of course
  a bit more complex
- it's not yet clear, whether it would be possible to access configMaps or
  secrets in the target project (at least was not possible now)
- not clear how/when would the dynamic parameters be supported
- not clear so far how to test this all
