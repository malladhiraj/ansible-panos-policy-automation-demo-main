# ansible-panos-policy-automation-demo
This repository gives a demo of using the paloaltonetworks.panos_policy_automation collection to do policy lookups and creation
specifcally within **Ansible Automation Platform.**

## AAP Setup

### Execution Enviornment 

Within AAP configure an Execution environment. This collection uses the same image as the main ansible collection.

**image**: ghcr.io/paloaltonetworks/panos_policy_automation-rhel9:latest

### Configure Credentials

Add a "network" type credential login for your device, using the Panorama username and password.

![img_1.png](docs/img/img_1.png)

### Configure a project

Create your repository that will store your Policy Files as well as your top level playbooks. You can
use this repo as a PoC/starting point if you like. You must create your own repository to specify
your preset policy files.

SCM URL: **https://github.com/adambaumeister/ansible-panos-policy-automation-demo**

### Create a job template for POLICY LOOKUP

Create a job template, ensuring you select the execution environment and project you just created. Ensure
'prompt on launch' is selected for Extra Variables.

![img_2.png](docs/img/img_2.png)

Example **Extra Variables**
```yaml
policy_creation_source_ip: 10.10.199.5
policy_creation_destination_ip: 10.10.200.20
lookup_policy_application: mysql
lookup_policy_destination_port: 3306
```

### Create a job template for CREATING POLICY

Same as above but choose your `create_policy` playbook.

Example **Extra Variables**
```yaml
policy_creation_source_ip: 10.10.199.5
policy_creation_destination_ip: 10.10.200.20
lookup_policy_application: mysql
lookup_policy_destination_port: 3306
policy_creation_policy_files:
  - policy_files/web_to_database.yaml
```


