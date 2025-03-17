👋 Introduction to confinguring private automation hub
===
#### Estimated time to complete: *45 minutes*<p>
In this section you will configure your private automation hub using the code provided that is ***missing some critical values/information*** that you will have to fill in yourself, based on the requirements and looking at readme's for the roles.

This lab uses `ansible-navigator` and has been tested against v3.4.1. It should be pre-installed on your machine.

The screen on the left shows the login screen. You can log in with the following credentials and then continue on to the tasks:

* Username: `admin`
* Password: `ansible123!`

A Gitea server is also provided to use with the same password as above, with the user account `ansible`. It is accessable using `https://gitea-8443-$INSTRUQT_PARTICIPANT_ID.env.play.instruqt.com/ansible/`

Further documentation for those who are interested to learn more see:

- [ansible navigator docs](https://ansible-navigator.readthedocs.io/en/latest/installation/#install-ansible-navigator)

☑️ Task 1 - Create Collection Repositories and Remotes
===
Create a file `group_vars/all/ah_repositories.yml` to create the list of community repositories and their remote counterpart.

```yaml
---
ah_collection_remotes:
  - name: community-infra
    url: https://galaxy.ansible.com/
    requirements:
      - name: infra.ee_utilities
        version: ">=3.1.0"
      - name: infra.aap_utilities
        version: ">=2.4.0"
      - name: containers.podman
        version: ">=1.13.0"
      - name: awx.awx
        version: ">=24.3.0"
      - name: community.general
        version: ">=8.6.0"
      - name: infra.ah_configuration
        version: ">=2.0.0"
      - name: infra.controller_configuration
        version: ">=2.7.0"
      - name: infra.eda_configuration
        version: ">=1.0.0"

ah_collection_repositories:
  - name: community-infra-repo
    description: "description of community-infra repository"
    pulp_labels:
      pipeline: "approved"
    distribution:
      state: present
    remote: community-infra

ah_configuration_collection_repository_sync_async_delay: 5
ah_configuration_collection_repository_sync_async_retries: 150
...
```

Further documentation for those who are interested to learn more see:

- [installing collections using cli](https://docs.ansible.com/ansible/devel/user_guide/collections_using.html#collections)
- [using collections in AAP](https://docs.ansible.com/ansible-tower/latest/html/userguide/projects.html#collections-support)


☑️ Task 2 - Create User
===

Create a file `group_vars/all/ah_users.yml` make sure this user has `is_superuser` set to `true` and their `password` is set to `"{{ ah_token_password }}"`.

```yaml
---
ah_token_username: "ah_token_user"
ah_users:
  - username: "{{ ah_token_username }}"
    groups:
      - "admin"
    append: true
    state: "present"
...
```

Further documentation for those who are interested to learn more see:

- [ah config users](https://github.com/ansible/galaxy_collection/blob/devel/roles/user/README.md)



☑️ Task 3 - Create a group
===

Create a file `group_vars/all/ah_groups.yml` and add `ah_groups` list with one (1) item in it with the `name` of `admin` and `state` is `present`. Do not set permisions.
If you need more information follow the documentation link below.
- [ah config groups](https://github.com/ansible/galaxy_collection/blob/devel/roles/group/README.md)

☑️ Task 4 - Create a playbook to configure hub
===

Create a playbook `playbooks/hub_config.yml` add in the `user` role name and the `collection_remote` role name in the tasks.

```yaml
---
- name: Configure private automation hub after installation
  hosts: execution
  gather_facts: false
  vars_files:
    - "../vault.yml"
  tasks:
    - name: Include group role
      ansible.builtin.include_role:
        name: infra.ah_configuration.group

    - name: Include user role
      ansible.builtin.include_role:
        name: infra.ah_configuration. # Insert Role Name here

    - name: Include collection remote role
      ansible.builtin.include_role:
        name: infra.ah_configuration. # Insert Role Name here

    - name: Include collection repository role
      ansible.builtin.include_role:
        name: infra.ah_configuration.collection_repository

    - name: Include collection repository role
      ansible.builtin.include_role:
        name: infra.ah_configuration.collection_repository_sync
...
```
If you need more information follow the documentation link below.
- [ah config groups](https://github.com/ansible/galaxy_collection/blob/devel/roles/group/README.md)

☑️ Task 5 - Run the playbook
===
The next step is to run the playbook, for demonstration purposes we are going to show how to get the Execution Environment(EE) that was built in the previous step and run the playbook.

Login to the automation hub using the podman login command. This will ask for a user:pass. After authenticating pull the config_as_code image.

Use the username: **'admin'** and the password for your account as listed at the top of this assignment.

```console
podman login --tls-verify=false hub.$INSTRUQT_PARTICIPANT_ID.instruqt.io
```
```console
podman pull --tls-verify=false hub.$INSTRUQT_PARTICIPANT_ID.instruqt.io/config_as_code:latest
```

Ansible navigator takes the following commands.
The options used are

|CLI Option|Use|
|:---:|:---:|
|`eei`|execution environment to use.|
|`i`|inventory to use.|
|`pa`|pull arguments to use, in this case ignore tls.|
|`m`|which mode to use, defaults to interactive.|

Use these options to run the playbook in the execution environment.

```console
ansible-navigator run playbooks/hub_config.yml --eei hub.$INSTRUQT_PARTICIPANT_ID.instruqt.io/config_as_code -i inventory.yml -l execution --pa='--tls-verify=false' -m stdout  --penv INSTRUQT_PARTICIPANT_ID
```
While the playbook is running you can go to the Automation Hub tab and peak at the Task Management to see the repositry syncing process
![task management](https://play.instruqt.com/assets/tracks/ruhjcwssqojn/f0aab6343c1db582036d1cb157a6f106/assets/image.png)

☑️ Task 6 - See the Results
===

Navigate to the Hub Tab, login with the provided passwords
* Username: `admin`
* Password: `ansible123!`

In each section on the left side you should find the changes you have made
Repository:
![Repository](https://play.instruqt.com/assets/tracks/ruhjcwssqojn/b3ee2fe044c2821b0b35e2d0c2cb06ee/assets/image.png)
User:
![user](https://play.instruqt.com/assets/tracks/ruhjcwssqojn/56c1cd58c860a64384e018a24a5252b7/assets/image.png)

✅ Next Challenge
===
Press the `Next` button below to go to the next challenge once you’ve completed the tasks.

🐛 Encountered an issue?
====

If you have encountered an issue or have noticed something not quite right, please [open an issue](https://github.com/ansible/instruqt/issues/new?labels=Introduction-to-AAP-config-as-code&title=Issue+with+Intro+to+AAP+config+as+code+slug+ID:+hub_exercise2&assignees=sean-m-sullivan).

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
