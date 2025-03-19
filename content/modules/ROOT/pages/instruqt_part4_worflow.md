👋 Using the Collection with Workflows
===
#### Estimated time to complete: *70 minutes*<p>

One of the goals of the Automation Controller Collection and the infra.controller_configuration collection is to allow users to easily create objects in Ansible Automation Platform. In service of that, we will explore how to codify workflows and deploy them on the automation controller.

The screen on the left shows the login screen. You can log in with the following credentials and then continue on to the tasks:

* Username: `admin`
* Password: `ansible123!`

A Gitea server is also provided to use with the same password as above, with the user account `ansible`. It is accessable using `https://gitea-8443-$INSTRUQT_PARTICIPANT_ID.env.play.instruqt.com/ansible/`

☑️  Overview
===
First, let’s take a look at the following workflow:
![workflow](https://play.instruqt.com/assets/tracks/ruhjcwssqojn/9d4868225f61545df0466dcead10de68/assets/image.png)


Each node represents an action, such as an inventory update, project update or the running of a job template. Other options can also be specified (see the docs for detail), and each node is linked to one or many other nodes. The common practice is to create the workflows inside of the automation controller GUI, but with the Collections, it is possible to manage these workflows through Ansible Playbooks.

## Structure
To start lets explore how each node is structured in code. Each entity in the workflow_nodes list describes a single node. Here is a breakdown of what each of the fields in the node represent:

```yaml
---
  - identifier: An identifier that is unique within its workflow.
    related:
      always_nodes: Dict List of nodes that always execute listed by identifiers
        - identifier: X
      failure_nodes: Dict List of nodes that execute on failure listed by identifiers
      success_nodes: Dict List of nodes that execute on success listed by identifiers
    unified_job_template:
      name: Name of Job Template that will be run
      organization:
        name: Name of Organization the Job template is in
      type: Type of node, this is required
...

```
The type can be one of job_template, project, inventory_source, workflow_approval.

### Inventory Sources
Unified Job template descriptions for inventory sources are slightly different to the others

```yaml
---
unified_job_template:
  inventory:
    organization:
      name: config_as_code ← Organization that the inventory source is in
  name: controller_config_source ← Name of inventory source
  type: inventory_source ← Type
...

```
Note that the organization is a sub variable of the inventory and is the organization that the inventory belongs to, not the workflow.


### Approval Nodes
Approval nodes do not have organizations

```yaml
---
unified_job_template:
  description: Approval node to continue job
  timeout: 900  ← Timeout before the Approval fails
  type: workflow_approval ← Type
  name: Approve to config controller ← Name that shows on node
...

```
### Project updates and Job templates
Project updates and Job templates share the same model
```yaml
---
unified_job_template:
  name: Demo Project ← Name of the Project or Job Template
  organization:
    name: Default ← Organization the Project or Job Template is in
  type: project ← Type
...

```
Note that the type can be either project or job_template

##Changing the workflow
By modifying the variable that represents the workflow, we can add additional nodes, remove or remap existing nodes, or even change the job template a node runs, amongst other things.

An example of removing a node:
```yaml
---
- identifier: Demo Job Template
  unified_job_template:
    name: Demo Job Template
    organization:
      name: Satellite
    type: job_template
  state: absent
...
```
It still requires a valid unified job template, but it will delete the node from the workflow.

☑️ Task 1 - Putting it together
===

The above instructions are building blocks. Time to put them to use, below is the start of the workflow illustrated above. Using what you know, recreate it in the controller
![workflow](https://play.instruqt.com/assets/tracks/ruhjcwssqojn/9d4868225f61545df0466dcead10de68/assets/image.png)
```yaml
---
workflow_job_templates:
  - name: Configuration Workflow
    workflow_nodes:
      - identifier: Sync Inventory
        unified_job_template:
          inventory:
            name: config_as_code
            organization:
              name: config_as_code
            type: inventory
          name: controller_config_source
          type: inventory_source
        related:
          always_nodes:
            - identifier: Configure Hub
...

```

Use the following `playbooks/workflow_config.yml` playbook to just run workflows
```yaml
---
- name: Playbook to configure workflows
  hosts: all
  vars_files:
    - "../vault.yml"
  connection: local
  roles:
    - infra.controller_configuration.workflow_job_templates
...

```
With this console command using tags:
```console
ansible-navigator run playbooks/workflow_config.yml --eei hub.$INSTRUQT_PARTICIPANT_ID.instruqt.io/config_as_code -i inventory.yml -l execution --pa='--tls-verify=false' -m stdout  --penv INSTRUQT_PARTICIPANT_ID --tags workflow_job_templates
```

Further documentation for creating workflows can be found here:
- [workflow_job_templates role](https://github.com/redhat-cop/controller_configuration/tree/devel/roles/workflow_job_templates)
- [Automation controller workflow deployment as code - Ansible Blog](https://www.ansible.com/blog/automation-controller-workflow-deployment-as-code)


☑️ Task 2 - Exporting an existing workflow
===
Create the next  playbook `playbooks/export_workflow.yml`, To extract the information from the server about a specific workflow. The awx.awx.export or the ansible.controller module can also be used to extract other objects from the server as well.
```yaml
---
- name: Export Workflow
  hosts: automationcontroller
  gather_facts: false
  environment:
     CONTROLLER_HOST: "{{ controller_hostname }}"
     CONTROLLER_USERNAME: "{{ controller_username }}"
     CONTROLLER_PASSWORD: "{{ controller_password }}"
     CONTROLLER_VERIFY_SSL: "{{ controller_validate_certs }}"

  tasks:
    - name: Export workflow job template
      awx.awx.export:
        workflow_job_templates: Configuration Workflow
      register: export_results
	    delegate_to: localhost

    - debug:
        var: export_results

    - name: Export workflow job template to file
      copy:
        content: "{{ export_results.assets | to_nice_yaml( width=50, explicit_start=True, explicit_end=True) }}"
        dest: group_vars/all/workflow_export.yaml
...

Once created, run the playbook
```
Run the playbook with the following command
```console
ansible-navigator run playbooks/export_workflow.yml --eei hub.$INSTRUQT_PARTICIPANT_ID.instruqt.io/config_as_code -i inventory.yml -l automationcontroller --pa='--tls-verify=false' -m stdout  --penv INSTRUQT_PARTICIPANT_ID
```

Notice that the exported workflow has a lot more meta information that was not needed for configuration. Many of these are fields that contain options for tweaking how it runs.

✅ Next Challenge
===
Press the `Next` button below to go to the next challenge once you’ve completed the tasks.

🐛 Encountered an issue?
====

If you have encountered an issue or have noticed something not quite right, please [open an issue](https://github.com/ansible/instruqt/issues/new?labels=Introduction-to-AAP-config-as-code&title=Issue+with+Intro+to+AAP+config+as+code+slug+ID:+workflow_exercise4&assignees=sean-m-sullivan).

<style type="text/css" rel="stylesheet">
  .lightbox {
    display: none;
    position: fixed;
    justify-content: center;
    align-items: center;
    z-index: 999;
    top: 0;
    left: 0;
    right: 0;
    bottom: 0;
    padding: 1rem;
    background: rgba(0, 0, 0, 0.8);
    margin-left: auto;
    margin-right: auto;
    margin-top: auto;
    margin-bottom: auto;
  }
  .lightbox:target {
    display: flex;
  }
  .lightbox img {
    /* max-height: 100% */
    max-width: 60%;
    max-height: 60%;
  }
  img {
    display: block;
    margin-left: auto;
    margin-right: auto;
  }
  h1 {
    font-size: 18px;
  }
    h2 {
    font-size: 16px;
    font-weight: 600
  }
    h3 {
    font-size: 14px;
    font-weight: 600
  }
  p span {
    font-size: 14px;
  }
  ul li span {
    font-size: 14px
  }
</style>
