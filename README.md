# ansible-github-deploy-key

Role to automatically change your deploy key, therefore ensuring a fresh key every time. For the security-minded, or those who just want to totally forget about the need to setup deploy keys. I got bored of managing them so made them go away.

## Theory of use

Leaving SSH keys on boxes is generally unnecessary beyond the lifespan of a code deploy. They are there purely to get access to your private repository, fetch the code down, and then afterwards they are redundant. However, unless these are removed and/or revoked afterwards, then they need to be managed - either in vaults, or via the Github control panel.

_Yawn._

This role takes care of that. It creates a key locally on the ansible client (which it promptly deletes), and updates the Github API with the public key. The key in use is therefore only in existence for the life of the playbook run, and useless thereafter. An optional handler also wipes it from Github when the deploy is over.

## Example usage
```
---
- hosts: all
  vars:
    github_deploy_api_token: "{{ lookup('env', 'GITHUB_API_TOKEN') }}"
  roles:
    - { role: shrikeh.github-deploy-key }
    - { role: my-code-deploy }    
...
```
Note that this role specifically does *not* do the following things:
- populate the private key to a file
- delete the key off the host machine (it doesn't know where you saved it to)
- automatically clear the key from the Github API (it doesn't know when you've finished with it)

The private key is available in the host fact variable `github_deploy_ssh_private_key`. An example of use in your own playbook is below:

```YAML
- name: Create a temporary directory for the key
  command: 'mktemp -d'
  register: deploy_code_key_dir_create

- name: Set a fact for the directory path
  set_fact:
    deploy_code_key_dir: "{{ deploy_code_key_dir_create.stdout }}"

- name: Create a temporary private key file
  lineinfile:
    create: yes
    state: present
    line: "{{ github_deploy_ssh_private_key }}"
    dest: "{{ deploy_code_key_dir }}/key"
    mode: 0600
```

You can then later use Ansible's [git] module to deploy your code and then revoke the key afterwards:

```YAML
---
- name: Get the code from Github
  git:
    repo: 'git@github.com:shrikeh/ansible-github-deploy-key.git'
    dest: /path/to/deploy/dir
    key_file: "{{ deploy_code_key_dir }}/key"
  notify:
    - Revoke Github deploy key
...
```

It is recommended you create your own handler to wipe the private key on the host afterwards:
```YAML
---
# handlers/main.yml

- name: Delete the deploy key temporary directory
  file:
    path: "{{ deploy_code_key_dir }}"
    state: absent
...
```

## Requirements

The token specified in `github_deploy_api_token` must have the [minimum permissions][github_permissions] to create (and optionally delete) a deploy key. These are:
  - repo
  - admin:public_key

## Variables

`github_deploy_local_delegate`

Used for `delegate_to`. Default is `127.0.0.1`

`github_deploy_ssh_keygen_executable`

Specify where the `ssh-keygen` binary is. It defaults to simply `ssh-keygen`, assuming that it is in the `PATH`.

`github_deploy_keyscan_enable`

Specify if you wish the role to run ssh-keyscan against Github. The scan is delegated locally and specified with `run_once` for efficiency. By default this is enabled, and allows you to then populate an ssh_known_hosts file with the contents of `github_deploy_keyscan`. Setting this to false will skip the keyscan. 


[git]: https://docs.ansible.com/ansible/git_module.html "Ansible Git module documentation"
[github_api]: https://developer.github.com/v3/ "Github API documentation"
[github_permissions]: https://help.github.com/articles/repository-permission-levels-for-an-organization/ "Github permissions for an organisation"
