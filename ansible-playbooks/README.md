# ansible-playbooks

- [Requirements](#requirements)
- [Installation](#installation)
  - [Needed changes to the OpenShift Container Platform](#needed-changes-to-the-openshift-container-platform)

- [Known Limitation](#known-limitation)

---

## Requirements
* An App Host (Edge Gateway) for `fn_ansible` app
* A Linux machine for running ansible playbooks
  * You may use App Host as a Linux machine if you can add SSH public key. `fn_ansible` on App Host does not support running playbooks inside the container, so you need SSH connection from the App Host container to a Linux machine to run playbooks. Default `fn_ansible` image does not come with SSH password support so SSH key setup is needed.
  

---

## Installation
Please follow the `fn_ansible` app installation steps on the App Host. There is nothing special for the app.config settings.

You'll need to obtain SOAR API key and secret (to attach a log file to an incident), as well as SSH private and public keys (for ansible to work). If you are pulling script log from OpenShift Container Platform, service account token is also required (to access OpenShift API).

### Files to add to the fn_ansible

The files needed by this package is the following.

| File Name                    | File Path                          | File Type  | Remarks                                                      |
| ---------------------------- | ---------------------------------- | ---------- | ------------------------------------------------------------ |
| hosts                        | /var/rescircuits/ansible/inventory | Plain Text | Host information which runs ansible playbooks                |
| ssh_key                      | /var/rescircuits/ansible/env       | Plain Text | SSH private key                                              |
| variables.yml                | /var/rescircuits/ansible/project   | YAML       | Configuration Variables                                      |
| secrets.yml                  | /var/rescircuits/ansible/project   | YAML       | API key and secret                                           |
| multipart.j2                 | /var/rescircuits/ansible/project   | Plain Text | Template file to implement multipart POST, as ansible native multipart support requires ansible 2.10 (fn_ansible comes with older version) |
| ocp_cases_log_downloader.yml | /var/rescircuits/ansible/project   | YAML       | Ansible playbook to pull a log from OpenShift Container Platform (called from SOAR playbook) |
| soar_log_downloader.yml      | /var/rescircuits/ansible/project   | YAML       | Ansible playbook to pull a log from SOAR appliance (called from SOAR playbook) |

### hosts File

The following is the example.

```yaml
apphost ansible_host=192.168.65.4 ansible_user=integration
soar-server ansible_host=192.168.65.7 ansible_user=resadmin
```

First column is the name referenced from the playbooks and by default `apphost` is used for *ocp_cases_log_downloader.yml* and `soar-server` is used for *soar_log_downloader.yml*.

No root privilege is required for `apphost`.

In regards to `soar-server`, root privilege is required to read the log file.
If `ansible_user` is not *root* but a user who becomes *root* (such as *resadmin*), set the `soar_script_sudo_required`  parameter in *variables.yml* to `yes`.
As fn_ansible does not come with SSH password feature, NOPASSWD setting will be required.

If you change the host name in the hosts file, you'll also need to update `hosts` parameter value in *ocp_cases_log_downloader.yml* and *soar_log_downloader.yml* files.

### ssh_key File

Generate a SSH private and public key pair on any Linux machine.

Then copy the private key to the `fn_ansible` and append the public key to the target account on the Linux machine which runs ansible playbooks. Do not protect the private key with password.

### variables.yml File

Define the values specific to your environment.

To get `openshift_api_uri`, first `oc login` and then issue `oc whoami --show-server` command.

For CP4S SOAR, `soar_server_uri` is usually the CP4S UI URI prefixed by `cases-rest.`.

`soar_org_id` is a number such as `201`, not a SOAR org name. You can see this at Organization tab accessible from `≡ > Case Management (setting) > Permissions and access`.

If you experience a security warning, set the `validate_certs` to false.

```yaml
# Ansible native multipart support requires ansible 2.10
log_attach_template_path: "multipart.j2"

# CP4S OCP Variables
openshift_api_uri: "https://your_openshift_api_server:port"
cp4s_namespcae: "your_cp4s_namespace_name"
cases_log_container_name: "cases-scripting-log-tailer"
# CP4S SOAR Variables
soar_server_uri: "https://cases-rest.your_cp4s_server"
soar_org_id: 201

# Stand alone RHEL Variables
soar_script_log_path: "/var/log/resilient-scripting/resilient-scripting.log"
soar_script_sudo_required: yes

# Common
validate_certs: false
```

### secrets.yml File

You need to obtain SOAR API key and secret for this integration.

If you are pulling script log from OpenShift Container Platform, service account token is also required. See [Needed changes to the OpenShift Container Platform](#needed-changes-to-the-openshift-container-platform) for more detail.

```yaml
# OpenShift Secrets
openshift_sa_token: "your_openshift_service_account_token"

# SOAR Secrets
soar_api_key_id: "your_soar_api_key_id"
soar_api_key_secret: "your_soar_api_key_secret"
```

### Needed changes to the OpenShift Container Platform

#### Create a Service Account

CP4S SOAR log is inside the *isc-cases-application* container so we need to access the OpenShift Container Platform API to get it.

As the login user's token changes frequently, we'll need a service account on the platform.

To create a new service account in the current project: (I suppose you are in CP4S namespace such as `cp4s`)

```shell
oc create sa <service_account_name>
```

To view the token name for the service account:

```shell
oc describe sa <service_account_name>
```

Token name is at the `Tokens:` part of the output.

Once the token name is identified, you can get the secret by the following command:

```shell
oc describe secret <sa-token-name>
```

A long string at the bottom of the output is the service account token.

#### Create a Local Role

Note: If you don't want to create a local role inside product namespace, you can use the existing role `cluster-reader` instead. The role has much more privilege than the role we're creating, but if your OpenShift Container Platform is dedicated to the CP4S purpose, that'll be OK.

To create a local role (example: `custom-cp4s-pod-reader` below) under the CP4S namespace (example: `cp4s` below):

```shell
oc create role custom-cp4s-pod-reader --verb=get,list --resource=pods,pods/log -n cp4s
```

You can check the role you've just created:

```shell
oc describe role custom-cp4s-pod-reader
Name:         custom-cp4s-pod-reader
Labels:       <none>
Annotations:  <none>
PolicyRule:
  Resources  Non-Resource URLs  Resource Names  Verbs
  ---------  -----------------  --------------  -----
  pods/log   []                 []              [get list]
  pods       []                 []              [get list]
```

Then bind the service account to the created role: (`cp4s` is the namespace in this example)

```shell
oc adm policy add-role-to-user custom-cp4s-pod-reader -z <service_account_name> --role-namespace=cp4s -n cp4s
```

Note: If you chose to use `cluster-reader` role instead, you'll still need to bind the service account to the cluster role:

```shell
oc adm policy add-cluster-role-to-user cluster-reader -z <service_account_name>
```

## Known Limitation

Unlike app.config, there is no easy way to hide secrets in the yaml file.
