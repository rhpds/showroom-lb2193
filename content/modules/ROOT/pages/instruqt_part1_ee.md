👋 Introduction to automation controller
===
#### Estimated time to complete: *30minutes*<p>

Welcome to the **Introduction to automation controller**.

In this section we will show you step by step how to build an Execution Environment (EE) from code. You will also be publishing this EE to your own private automation hub which you will use in the rest of the tasks.

The screen on the left shows the login screen. You can log in with the following credentials and then continue on to the tasks:

* Username: `admin`
* Password: `ansible123!`

A Gitea server is also provided to use with the same password as above, with the user account `ansible`. It is accessable using `https://gitea-8443-$INSTRUQT_PARTICIPANT_ID.env.play.instruqt.com/ansible/`

☑️ Task 1 - Initial Install
===
Open up the terminal by right clicking the files section on the left, and Open in Integrated Terminal

Install our ee_utilities collection and containers.podman using `ansible-galaxy` command. Make sure the versions are correct.

```console
ansible-galaxy collection install infra.ee_utilities:3.1.2 infra.controller_configuration:2.5.1 containers.podman:1.10.3 community.general:7.3.0
```

Further documentation for those who are interested to learn more see:

- [installing collections using cli](https://docs.ansible.com/ansible/devel/user_guide/collections_using.html#collections)
- [using collections in AAP](https://docs.ansible.com/ansible-tower/latest/html/userguide/projects.html#collections-support)


☑️ Task 2 - Create our Variable Files
===

### **In the next few steps pay attention to the folder paths and make sure to put the files in the correct folders**

Set the variables to be used in the collections for use. These include hosts, usernames, and other variables that are reused in each role.

Create a file in this folder path
```yaml
group_vars/all/auth.yml
```
With the following content:

```yaml
---
controller_hostname: "{{ controller_host | default( 'control.' + lookup('ansible.builtin.env', 'INSTRUQT_PARTICIPANT_ID') + '.instruqt.io' )}}"
controller_username: "{{ controller_user | default('admin') }}"
controller_password: "{{ controller_pass }}"
controller_validate_certs: false
controller_request_timeout: 60

ah_host: "{{ ah_hostname | default( 'hub.' + lookup('ansible.builtin.env', 'INSTRUQT_PARTICIPANT_ID') + '.instruqt.io' )}}"
ah_username: "{{ ah_user | default('admin') }}"
ah_password: "{{ ah_pass }}"
ah_path_prefix: 'galaxy'  # this is for private automation hub
ah_validate_certs: false

ee_registry_username: "{{ ah_username }}"
ee_registry_password: "{{ ah_password }}"
ee_registry_dest: "{{ ah_host }}"

ee_base_registry: "{{ ah_host }}"
ee_base_registry_username: "{{ ah_username }}"
ee_base_registry_password: "{{ ah_password }}"
...

```


Further documentation for those who are interested to learn more see:

- [more about group_vars](https://docs.ansible.com/ansible/latest/user_guide/intro_inventory.html#organizing-host-and-group-variables)



☑️ Task 3 - Create our Inventory File
===

Create your inventory file `inventory.yml`, You do not need to replace anything in this file.

```yaml
---
all:
  children:
    automationcontroller:
      hosts:
        control:
          ansible_host: "{{ 'control.' + lookup('ansible.builtin.env', 'INSTRUQT_PARTICIPANT_ID') + '.svc.cluster.local' }}"
    automationhub:
      hosts:
        hub:
          ansible_host: "{{ 'hub.' + lookup('ansible.builtin.env', 'INSTRUQT_PARTICIPANT_ID') + '.svc.cluster.local' }}"
    builder:
      hosts:
        control:
          ansible_host: "{{ 'control.' + lookup('ansible.builtin.env', 'INSTRUQT_PARTICIPANT_ID') + '.svc.cluster.local' }}"
    execution:
      hosts:
        localhost:
          ansible_connection: local
...
```

NOTE: These are hostnames do not have 'https://' or things will fail

Further documentation for those who are interested to learn more see:

- [more about inventories](https://docs.ansible.com/ansible/latest/user_guide/intro_inventory.html#inventory-basics-formats-hosts-and-groups)
- [how to use this source in AAP](https://docs.ansible.com/ansible-tower/latest/html/userguide/inventories.html#add-source)

☑️ Task 4 - Create our Vault
===
Create a vault file `vault.yml` and **YOU WILL NEED TO FILL THESE IN** with the correct passwords for each variable. Currently they are set to the description of what you should be updating them too.

```yaml
---
vault_pass: # CHANGE ME !!! the password to decrypt this vault you create this
machine_pass: 'ansible123!'
controller_pass: 'ansible123!'
ah_pass: 'ansible123!'
controller_api_user_pass: 'ansible123!'
ah_token_password: 'ansible123!'
student_account: 'ansible123!'
...

```

NOTE: the easiest way to do this is have all passwords be the provided password.


Create a `.password` file **We do not recommend you do this outside of lab environment** put your generated password in this file. Even though we are not committing this file into git because we have it in our ignore list, we do not recommend putting passwords in plain text ever, this is just to simplify/speed up the lab.

```text
Your_Generated_Password_In_.password_File
```

Create an `ansible.cfg` file to point to the .password file.

```ini
[defaults]
vault_password_file=.password
```

Encrypt vault with the password in the .password file

```console
ansible-vault encrypt vault.yml
```

Further documentation for those who are interested to learn more see:

- [ansible vaults](https://docs.ansible.com/ansible/latest/user_guide/vault.html)
- [vault with navigator](https://ansible.readthedocs.io/projects/navigator/faq/#how-can-i-use-a-vault-password-with-ansible-navigator)

☑️ Task 5 - Create our Playbook
===
Create a new playbook called `playbooks/build_ee.yml` and make the hosts use the group builder (which for this lab we are using automation hub, see note) and turn gather_facts on. Then add include role infra.ee_utilities.ee_builder

Note: this we would normally suggest being a small cli only server for deploying config as code and running installer/upgrades for AAP

```yaml
---
- name: Playbook to configure execution environments
  hosts: builder
  gather_facts: false
  connection: local
  vars_files:
    - "../vault.yml"
  tasks:
    - name: Include ee_builder role
      ansible.builtin.include_role:
        name: infra.ee_utilities.ee_builder

    - name: Add Execution Environment
      ansible.builtin.include_role:
        name: infra.controller_configuration.execution_environments
...

```

Further documentation for those who are interested to learn more see:

- [include vs import](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/include_role_module.html)
- [ee_builder role](https://github.com/redhat-cop/ee_utilities/tree/main/roles/ee_builder)

☑️ Task 6 - Create our EE Definition
===
Create a file `group_vars/all/ah_ee_list.yml` where we will create a list called `ee_list` that has 4 variables per item which are:

- `name` this is required and will be what the EE image will be called
- `bindep` this is any system packages that would be needed
- `python` these are any python modules that need to be added through pip (excluding ansible)
- `collections` any collections that you would like to be built into your EE image

which the role will loop over and for each item in this list it will create and publish an EE using the provided variables.

```yaml
ee_list:
  - name: "config_as_code"
    pull: always
    dependencies:
      galaxy:
        collections:
          - name: infra.controller_configuration
            version: 2.7.1
          - name: infra.ah_configuration
            version: 2.0.6
          - name: infra.ee_utilities
            version: 3.1.2
          - name: awx.awx
            version: 24.3.0
          - name: containers.podman
            version: 1.13.0
          - name: community.general
            version: 8.6.0

ee_base_image: "{{ ah_host }}/ee-minimal-rhel8:latest"
ee_image_push: true
ee_prune_images: false
ee_create_ansible_config: false
ee_pull_collections_from_hub: false
ee_create_controller_def: true
...
```
=======

Further documentation for those who are interested to learn more see:

- [YAML lists and more](https://docs.ansible.com/ansible/latest/reference_appendices/YAMLSyntax.html)
- [builder role documentation](https://github.com/redhat-cop/ee_utilities/blob/main/roles/ee_builder/README.md#build-argument-defaults)


☑️ Task 7 - Check your paths
===
Your folder structure should look like this, check the file structure to make sure files are in the right levels. Run the `tree` command to verify.


```yaml
    ├── ansible.cfg
    ├── group_vars
    │   └── all
    │       ├── ah_ee_list.yml
    │       └── auth.yml
    ├── inventory.yml
    ├── playbooks
    │   ├── build_ee.yml
    └── vault.yml
```



☑️ Task 8 - Run the Playbook
===

Run the playbook pointing to the recently created inventory file and limit the run to just builder to build your new custom EE and publish it to private automation hub.

```console
ansible-playbook -i inventory.yml -l builder playbooks/build_ee.yml
```

Further documentation for those who are interested to learn more see:

- [more on ansible-playbook](https://docs.ansible.com/ansible/latest/cli/ansible-playbook.html#ansible-playbook)


☑️ Task 9 - See the Results
===

Navigate to the Hub and Controller Tabs, login with the provided passwords
* Username: `admin`
* Password: `ansible123!`

In each applications respective Execution Environment section on the left side you should find the `config_as_code` Execution Environment.

![Controller EE](https://play.instruqt.com/assets/tracks/ruhjcwssqojn/655ee91a55ad3d68ce26b0058f87630f/assets/image.png)
![Hub EE](https://play.instruqt.com/assets/tracks/ruhjcwssqojn/4dcf4912728c72e4c0ba986de4546fbd/assets/image.png)

✅ Next Challenge
===
Press the `Next` button below to go to the next challenge once you’ve completed the tasks.

🐛 Encountered an issue?
====

If you have encountered an issue or have noticed something not quite right, please [open an issue](https://github.com/ansible/instruqt/issues/new?labels=Introduction-to-AAP-config-as-code&title=Issue+with+Intro+to+AAP+config+as+code+slug+ID:+ee_utils_exercise1&assignees=sean-m-sullivan).

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
