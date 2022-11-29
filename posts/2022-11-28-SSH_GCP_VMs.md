---
title: "SSH into GCP VMs"
description: "Add SSH keys to GCP VMs to allow Ansible access"
toc: true
draft: false
date: '2022-11-28'
categories:
- sql
- GCP
comments:
  giscus:
    repo: stantonius/home
---


## Objective

Configure a GCP VM to accept SSH connections via the traditional (ie. not `gcloud`) command line method so that we can use tools like Ansible which require SSH.

## Background

By default, when I create VMs using Terraform, I set the GCP VM option of `enable-oslogin` to TRUE. This means I can use the `gcloud` SDK to connect to the VM without having to worry about managing SSH keys. More information can be found [here](https://cloud.google.com/compute/docs/oslogin).

The important this to realize is that without additional configuration, we cannot SSH into a OS Login-enabled VM using the standard `ssh user@ip_address`

## Methods to add SSH keys to VMs

As seen in [the docs](https://cloud.google.com/compute/docs/instances/ssh#third-party-tools_1), there are two approaches for adding SSH keys to VMs.

### Metadata

This is the traditional approach where you add a pulic key to the `~/.ssh/authorized_keys` file of a VM. This is the approach used when configuring `cloud-init`.

### OS Login

The GCP-managed SSH configuration that allows us to use the `gcloud` SDK to SSH into a VM. The advantage in normal circumstances is that GCP manages all of the SSH keys - all we need to do is have `gcloud` authorize us on our local machine. 

It is important to note that when using OS Login, the **username is set for us by default**. The format for the username replaces special characters with underscores. For example, if your email was `james@example.com`, your OS Login username for the VM would be `james_example_com`.

> When OS Login is enabled, Compute Engine refuses connections from SSH keys that are stored in metadata.

## Add keys to OS Login VMs

If we have chosen to use OS Login, we need to use `gcloud` to [add our own keys to the OS Login-enabled VM](https://cloud.google.com/compute/docs/connect/add-ssh-keys#os-login). Note that we are not just adding SSH keys to a single VM, but rather we are adding SSH keys for every OS Login-enabled VM in our project. As such, we need to make sure we are authenticated with the correct account using `gcloud`

To add an SSH key for our account to the project, we need to run the `gcloud`  command

```bash
gcloud compute os-login ssh-keys add \
    --key-file=KEY_FILE_PATH \
    --project=PROJECT \
    --ttl=EXPIRE_TIME
```

Now we can SSH into an OS Login-enabled VM using the command: `ssh james_example_com@ip_address -i path_to_ssh_key`