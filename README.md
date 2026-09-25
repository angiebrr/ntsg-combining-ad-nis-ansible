# Combining NIS and AD with Ansible

> [!WARNING]
> Archived and no longer maintained; kept for reference. Written in 2016 for CentOS 6 and 7, which are both end-of-life.

- [Combining NIS and AD with Ansible](#combining-nis-and-ad-with-ansible)
  - [Overview](#overview)
    - [What this playbook does](#what-this-playbook-does)
  - [Using it](#using-it)
    - [Variable files](#variable-files)
      - [all.yml](#allyml)
      - [vault.yml](#vaultyml)
      - [hosts](#hosts)
    - [Provision the machine(s) in hosts](#provision-the-machines-in-hosts)
    - [Running the playbook](#running-the-playbook)

## Overview

An Ansible playbook that joins CentOS 6 and 7 servers to Active Directory for authentication while keeping NIS for user and group identities.

I wrote this in 2016 as the Linux sysadmin for NTSG, a research group at the University of Montana. Our servers were full of NIS UIDs and GIDs we couldn't easily migrate, but we wanted people to log in with their AD password so researchers only had one password to remember. Setting that up by hand took a while and was easy to get wrong if you missed a step, so I automated it.

**Tech:** Ansible, SSSD, Kerberos, adcli, NIS (ypbind), chrony, CentOS 6/7

### What this playbook does

- Makes each host an NIS client
- Joins it to the AD domain with `adcli`
- Configures SSSD to take identity from NIS and authentication from AD
- Creates home directories on first login with oddjob-mkhomedir
- Backs up every file it changes to `/etc/backups`

A later CentOS 7 version that uses `realmd` is in [configuring-ad-in-ansible](https://github.com/angiebrr/configuring-ad-in-ansible).

## Using it

### Variable files

You will need to do a little bit of setup before using this playbook. The first thing to do is to make sure that you have the following variable and host files filled out:

- `group_vars/all.yml`
- `group_vars/vault.yml`
- `hosts`

#### all.yml

This has the variables that all hosts will use for the ad, nis, and join roles. There is an example yml file that you template off of. This file also has descriptions of each variable so you know exactly what you are setting.

```yaml
# group_vars/all.yml.example

# ------------------------------------------------------------------------
# ANSIBLE SPECIFIC VARS
# ------------------------------------------------------------------------

# User that ansible will use to log in with
playbook_remote_user: username

# The default directory on the remote machine that backups will be moved to
back_up_dir: "/etc/backups"

# ------------------------------------------------------------------------
# NIS VARS
# ------------------------------------------------------------------------

# The NIS domain name (NOT fqdn)
nis_domain: nisdomain

# A list of addresses of NIS servers. You can either specify the master 
# NIS server, NIS slave server/s, or both.
nis_servers:
  - addr: 0.nis.domain.com
  - addr: 1.nis.domain.com

# ... (see group_vars/all.yml.example for more)

```

#### vault.yml

You will also need to create a vault.yml file with the following variables inside:

```yaml
# group_vars/vault.yml

vault_ad_user: <AD username>
vault_ad_pass: <AD password>
```

It is recommended that you create this file using `ansible-vault` like so:

```bash
$ ansible-vault create group_vars/vault.yml
```

For more information on using `ansible-vault`, see the Ansible documentation: [Ansible Vault](https://docs.ansible.com/ansible/latest/vault_guide/index.html)

#### hosts

This, as usual, contains the host information of the machines that will run this playbook (i.e. the inventory)

### Provision the machine(s) in hosts

Before running this playbook, you have to provision the machines in your hosts file in Active Directory in Windows, as I haven't found a way to automate this. All you need to do is add a computer under the AD domain and make it the same name as the hostname of the linux machine.

### Running the playbook

A typical run of the playbook will look something like this:

```bash
$ ansible-playbook -i hosts site.yml --ask-vault-pass
```

The `--ask-vault-pass` parameter will ask for the password to your vault-created file with the variables `vault_ad_user` and `vault_ad_pass`.

There are a couple of things to keep in mind when running this playbook: 
- Due to the necessity of using authconfig and adcli (which can only be called from the command module), running the playbook in its entirety will always return a "changed" of 3.
- Every changed file is backed up in place, so do not despair if something goes awry! The backups are, by default, in `/etc/backups`

Here is a list of files that *could be* changed/created by the playbook:
- /etc/sysconfig/network
- /etc/yp.conf
- /etc/nsswitch.conf
- /etc/chrony.conf
- /etc/ssh/sshd_config
- /etc/krb5.conf
- /etc/sssd/sssd.conf

The only files that will be completely 100% clobbered (as opposed to just changing a few lines) are:
- /etc/krb5.conf
- /etc/sssd/sssd.conf
