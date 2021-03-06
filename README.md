# Ansible Local Provisioner

Type: ansible-local

The Ansible local provisioner configures Ansible to run on the machine by Packer from local Playbook and Role files.  Playbooks and Roles can be uploaded from your local machine to the remote machine.  Ansible is run in [local mode](http://www.ansibleworks.com/docs/playbooks2.html#local-playbooks) via the ansible-playbook command.

## Install

Download and build Packer from source as described [here](https://github.com/mitchellh/packer#developing-packer).

Next, clone this repository into `$GOPATH/src/github.com/kelseyhightower/packer-provisioner-ansible-local`.  Then build the packer-provisioner-ansible-local binary:

```
go build -o /usr/local/packer/packer-provisioner-ansible-local \
plugin/provisioner-ansible-local/main.go
```

Now [configure Packer](http://www.packer.io/docs/other/core-configuration.html) to pick up the new provisioner:

```
{
  "provisioners": {
    "ansible-local": "/usr/local/packer/packer-provisioner-ansible-local"
  }
}
```

## Basic Example

The example below is fully functional and expects the configured Playbook file to exist relative to your working directory:

```JSON
{
  "builders": [{
    "type": "amazon-ebs",
    "access_key": "AKIAIOSFODNN7EXAMPLE",
    "secret_key": "wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY",
    "region": "us-east-1",
    "source_ami": "ami-05355a6c",
    "instance_type": "t1.micro",
    "ssh_username": "ec2-user",
    "ssh_timeout": "5m",
    "ami_name": "ansible-local-{{timestamp}}"
  }],
  "provisioners": [
    {
      "type": "shell",
      "inline": [
        "sleep 30",
        "sudo yum install ansible --enablerepo=epel --enablerepo=epel-testing -y"
      ]
    },
    {
      "type": "ansible-local",
      "playbook_file": "local.yml"
    }
  ]
}
```

You can also upload roles and additional Playbooks:

```JSON
{
  "provisioners": [
    {
      "type": "ansible-local",
      "playbook_file": "local.yml",
      "playbook_paths": [
        "playbooks/mysql.yml"
      ],
      "role_paths": [
        "roles/nodejs"
      ]
    }
  ]
}
```

And use them like this:

```YAML
# local.yml
---
- include: playbooks/mysql.yml
- hosts: 127.0.0.1
  sudo: yes
  connection: local
  tasks:
    - name: "ensure nginx is installed"
      yum: name="nginx" state=installed enablerepo="epel"
  roles:
    - nodejs
```

## Configuration Reference

The reference of available configuration options is listed below.

Required parameters:

 * `playbook_file` (string) - The playbook file to be executed by ansible. This file must exist on your local system and will be uploaded to the remote machine.

Optional parameters:

 * `playbook_paths` (array of strings) - This is an array of paths to playbook files on your local system. These will be uploaded to the remote machine under `staging_directory`/playbooks. By default, this is empty.
 * `role_paths` (array of strings) - This is an array of paths to role directories on your local system. These will be uploaded to the remote machine under `staging_directory`/roles. By default, this is empty.
 * `staging_directory` (string) - This is the directory where all the configuration of Ansible by Packer will be placed.  By default this is "/tmp/packer-provisioner-ansible-local".  This directory doesn't need to exist but must have proper permissions so that the SSH user that Packer uses is able to create directories and write into this folder. If the permissions are not correct, use a shell provisioner prior to this to configure it properly.

## Execute Command

By default, Packer uses the following command to execute Ansible:

    ansible-playbook {{.PlaybookFile}} -c local -i "127.0.0.1,"

