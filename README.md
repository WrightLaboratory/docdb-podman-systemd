# Containerized DocDB under Rootless Podman Systemd

## Prepare Ansible Management Node

Install Python using (`pyenv`)[https://github.com/pyenv/pyenv]:

```
pyenv install $(cat .python-version)

python -m venv ./.venv

source ./.venv/bin/activate

pip install -r requirements.txt
```

Clone this repository:

```
git clone docdb-podman-systemd
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
ansible-playbook -i localhost, --connection local docdb-secrets-generate.yml
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

Finally, execute `main.yml`:

```
export DOCDB_PROJECT_NAME="{{ DocDBProjectName }}"
export DOCDB_SHORTPROJECT_NAME="{{ DocDBShortProjectName }}"
export TARGET_FQDN="{{ FQDNOfDocDBServer }}"
export TARGET_USER="{{ NameOfUserWithAdminPrivilegesOnServer }}"

ansible-playbook --user="${TARGET_USER}" \
    --vault-id "$(id -un)@${HOME}/.ansible/vaultpassword" \
    -e "@${HOME}/.ansible/${DOCDB_SHORTPROJECT_NAME}-secrets.yml" \
    -i "${TARGET_FQDN}", \
    main.yml
```