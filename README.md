# Containerized DocDB under Rootless Podman Systemd

## Introduction

[DocDB](https://github.com/ericvaandering/DocDB.git) is a document management system developed by Eric Vaandering at Fermi Lab and used by the particle physics community to track the artifacts of a specific collaboration.

Its notion of a document extends beyond journal articles and their drafts to encompass meetings, symposia notes, slides, operational documentation, etc.

Since a particle physics project may span decades, it is important to implement a DocDB instance that assures durability and maintainability over its projected lifespan.

The need to upgrade the host operating system and the reliance on a specific server incurs a non-trivial amount of technical debt.

Containerizing a DocDB instance helps alleviate some this debt by freezing the operating system and runtime environments.

We have chosen Podman over Docker as its architecture lends itself to running applications with the least system privileges.
Furthermore, Podman with its support for Kubernetes YAML provides for a frictionless migration path from a single node deployment to a clustered Kubernetes enviromnent.

While the design is straightforward, the technical details of configuring the host system's support for rootless Podman is rather involved, especially under Redhat Enterprise 8 Linux and its derivative like AlmaLinux.

These details are sorted in the accompanying Ansible code in the [`ansible-roles`](https://github.com/WrightLaboratory/ansible-roles.git) repository.

## Prepare Ansible Management Node

Clone this repository:

```
git clone https://github.com/WrightLaboratory/docdb-podman-systemd.git docdb-podman-systemd

cd docdb-podman-systemd
```

Install Python using [`pyenv`](https://github.com/pyenv/pyenv).

```
pyenv install $(cat .python-version)

python -m venv ./.venv

source ./.venv/bin/activate

pip install -r requirements.txt
```

Pull the required Ansible roles from the the Wright Laboratory `ansible-roles` repository as a Git submodule:


```
cd docdb-podman-systemd

git submodule add  https://github.com/WrightLaboratory/ansible-roles.git roles

# Install required Ansible modules
ansible-galaxy install -r ./roles/requirements.yml
```

## Running Playbook

Generate Ansible vault file containing DocDB configuration and secrets:

```
export DOCDB_PROJECT_NAME="{{ DocDBProjectName }}"
export DOCDB_SHORTPROJECT_NAME="{{ DocDBShortProjectName }}"

ansible-playbook -i localhost, --connection local docdb-secrets-generate.yml
ansible-vault encrypt --vault-id $(id -un)@${HOME}/.ansible/vaultpassword ${HOME}/.ansible/${DOCDB_SHORTPROJECT_NAME}-secrets.yml
```

The output will look like the following:

```
Please provide the name of your research project or collaboration. (It may contain spaces): Dark Matter Anomaly Collaboration
Please provide an abbreviation of your research project or collaboration name: dmaco      
Please provide the contact email for this DocDb web instance: s.tilly@daystrom.org
Please provide a password to access the administrator only pages of this DocDB instance: SuperSekret31
Please provide a password for routine user access to the pages of this DocDB instance: sequence57Test
Please provide a password for read-only access to the pages of this DocDB instance: remote32Sock

PLAY [localhost] **********************************************************************************************************************************************

TASK [Gathering Facts] ****************************************************************************************************************************************

ok: [localhost]

TASK [check for existence of ~/.ansible/DarkMatterAnomaly-secrets.yml] ****************************************************************************************
ok: [localhost]

TASK [check for existence of ~/.ansible/vaultpassword] ********************************************************************************************************
ok: [localhost]

TASK [create random string and save it in mysql_root_password] ************************************************************************************************
ok: [localhost]

TASK [create random string and save it in mysql_password] *****************************************************************************************************
ok: [localhost]

TASK [display mysql password] *********************************************************************************************************************************
ok: [localhost] => {
    "msg": "Password generated for DocDB database admin DarkMatterAnomaly_admin: qBKwwNRZ_.w5"
}

TASK [create random string and save it in docdb_db_user_rw_passwd] ********************************************************************************************
ok: [localhost]

TASK [create random string and save it in docdb_db_user_ro_passwd] ********************************************************************************************
ok: [localhost]

TASK [create docdb secrets variable file in ~/.ansible on control node] ***************************************************************************************
changed: [localhost]

TASK [create default ansible vault password file (~/.ansible/vaultpassword) with correct permissons] **********************************************************
skipping: [localhost]

PLAY RECAP ****************************************************************************************************************************************************
localhost                  : ok=9    changed=1    unreachable=0    failed=0    skipped=1    rescued=0    ignored=0   
```

Prepare environment for running playbook.

```
export TARGET_FQDN="{{ FQDNOfDocDBServer }}"
export TARGET_USER="{{ NameOfUserWithAdminPrivilegesOnServer }}"
```

Generate an archive that contains an SSL certficate, key, and bundle file using this [repository](https://github.com/vbalbarin/cert-manager.git).
(Note, you use generate a self-signed certificate and key without a certificate authority chain but Chromium-based browser will not allow you to bypass the certificate error.)

Extract this archive to `./roles/nginx-revers-proxy/files`

```
mkdir -p ./roles/nginx-reverse-proxy/files
tar -xvf ${TARGET_FQDN}.gz -C ./roles/nginx-reverse-proxy/files
```

Execute playbook:

```
ansible-playbook --user="${TARGET_USER}" \
    --vault-id "$(id -un)@${HOME}/.ansible/vaultpassword" \
    -e "@${HOME}/.ansible/${DOCDB_SHORTPROJECT_NAME}-secrets.yml" \
    -i "${TARGET_FQDN}", \
    main.yml
```