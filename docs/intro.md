---
sidebar_position: 1
slug: /
---

# Hands-on: Maximo 7.x to MAS Manage Upgrade Lab 1261

## 1 Install Single Node OpenShift via APIs

### 1.1 Prepare token and secrets

#### 1.1.1 API token

1. Change directory.
```bash
cd 1261
```

2. Refresh provided token.
```bash
source ./refresh-token
```

3. Verify that you can access the API by running the following command:
```bash
curl -s https://api.openshift.com/api/assisted-install/v2/component-versions \
-H "Authorization: Bearer ${API_TOKEN}" \
| jq
```

:::note

Expected outcome

:::
```json
{
  "release_tag": "v2.33.1",
  "versions": {
    "assisted-installer": "registry.redhat.io/rhai-tech-preview/assisted-installer-rhel8:v1.0.0-353",
    "assisted-installer-controller": "registry.redhat.io/rhai-tech-preview/assisted-installer-reporter-rhel8:v1.0.0-431",
    "assisted-installer-service": "quay.io/app-sre/assisted-service:4b0d0eb",
    "discovery-agent": "registry.redhat.io/rhai-tech-preview/assisted-installer-agent-rhel8:v1.0.0-332"
  }
}
```

#### 1.1.2 Software (pull) secret

4. Software (pull) secret in JSON format has been made available to you in advance. Save it as a variable.

```bash
PULL_SECRET=$(cat ./pull-secret-escaped.json)
```

#### 1.1.3 Secure Shell (SSH) key

5. SSH key has been made available to you in advance. Save it as a variable.
```bash
SSH_KEY=$(cat ~/.ssh/id_ed25519.pub)
```

#### 1.1.4 Name your OpenShift

6. Save your assigned name as a variable. Replace XX with your number before executing the command.
```bash
export NAME=ocpwksXX
```

#### 1.1.5 OpenShift version

7.	At the time of this workshop, we are working with version 4.15. Save it as a variable.
```bash
export VERSION=4.15
```

### 1.2 Create the instance

1. Using the above saved values, create your instance.
```bash
RESPONSE=$(curl -s -X POST "https://api.openshift.com/api/assisted-install/v2/clusters" \
-H "Content-Type: application/json" \
-H "Authorization: Bearer ${API_TOKEN}" \
-d '{
  "name": "'"${NAME}"'",
  "high_availability_mode": "None",
  "openshift_version": "4.15",
  "base_dns_domain": "gym.lan",
  "ssh_public_key": "'"${SSH_KEY}"'",
  "pull_secret": "'"${PULL_SECRET}"'",
  "network_type": "OVNKubernetes",
  "schedulable_masters": true
}') ; echo $RESPONSE
```

2. Identify the clusterID.
```bash
CLUSTER_ID=$(echo $RESPONSE | jq -r '.cluster_networks[0].cluster_id') ; echo $CLUSTER_ID
```

3. Get Cluster Status
```bash
STATUS=$(curl -s -X GET "https://api.openshift.com/api/assisted-install/v2/clusters/${CLUSTER_ID}" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $API_TOKEN") ; echo $STATUS
```

4. Create Infra Env
```bash
export INFRA_ENV_ID=$(curl -s -X POST "https://api.openshift.com/api/assisted-install/v2/infra-envs" \
-H "Content-Type: application/json" \
-H "Authorization: Bearer ${API_TOKEN}" \
-d '{
  "ssh_authorized_key": "'"${SSH_KEY}"'",
  "pull_secret": "'"${PULL_SECRET}"'",
  "name": "'"${NAME}"'",
  "image_type": "full-iso",
 "cluster_id": "'"${CLUSTER_ID}"'",
 "openshift_version": "4.15"
}') ; echo $INFRA_ENV_ID
```

5. Get ISO URL
```bash
ISO_URL=$(echo $INFRA_ENV_ID | jq -r '.download_url')
```

6. Download ISO
```bash
wget -O ~/Downloads/${NAME}.iso $ISO_URL
```

### 1.3 Create VMware Virtual Machine

1. Create VM.
```bash
./govc_create_vm.sh
```

:::warning

You must wait few minutes here.

:::

### 1.3 Install SNO

1. After about a few minutes, install OpenShift.
```bash
source ./refresh-token
```
```bash
curl -X POST "https://api.openshift.com/api/assisted-install/v2/clusters/${CLUSTER_ID}/actions/install" \
-H "Authorization: Bearer ${API_TOKEN}"
```

2. Get Status
```bash
while [ "$STATUS" != "installed" ]; do
    STATUS=$(curl -s -X GET "https://api.openshift.com/api/assisted-install/v2/clusters/${CLUSTER_ID}" \
    -H "Authorization: Bearer ${API_TOKEN}" | jq -r .status)
    echo "Current Status: $STATUS"
    if [ "$STATUS" != "installed" ]; then
        # Wait for a while before checking again
        sleep 90
    fi
done
```

:::note

Expected outcome:
```
Current Status: preparing-for-installation
Current Status: installing
Current Status: installing
Current Status: finalizing
Current Status: null
```

The while loop times out when the token expires. Take the following actions when you see the `null` message...
1. Press Ctrl+c to exit the while loop.
2. Run `source ./refresh-token` 
3. Copy/paste the while loop command to start getting the status again.
:::



## ⏰ ~45 minutes

:::info

While we wait for the Openshift to finish the installation, we switch gears and work on Maximo customization and Maximo database migration. We will come back to OpenShift after the installation has been completed successfully.

Jump down below to section number 3.

:::

## 2 OpenShift Day two operations

### 2.1 OpenShift credentials

#### 2.1.1 Retrieve OpenShift credentials

1. Refresh your API token.
```bash
source ./refresh-token
```

2. Retrieve credentials.
```bash
curl -s -X GET "https://api.openshift.com/api/assisted-install/v2/clusters/${CLUSTER_ID}/credentials" -H "Authorization: Bearer ${API_TOKEN}" | jq
```

#### 2.1.2 Save credentials.

3. Copy/paste the retreived credentials to the passwords.txt file.

4. Replace xxxx-xxxx with the kubeadmin password.

### 2.2 MAS CLI

#### 2.2.1 Setup MAS CLI

5. Change directory
```bash
cd mas9
```

6. Start MAS CLI
```bash
podman run -it --pull always -v ${PWD}:/mas9:Z --name ibmmas quay.io/ibmmas/cli:10.9.1
```

7. Change directory
```bash
cd /mas9
```
```bash
cp /mascli/ansible-devops/common_vars/default_storage_classes.yml /mascli/ansible-devops/common_vars/default_storage_classes.yml.bak && mv default_storage_classes.yml /mascli/ansible-devops/common_vars/default_storage_classes.yml
```

### 2.3 Command line login to OpenShift

8. Copy/paste the `oc login ...` string from your passwords.txt file to login to your OpenShift from inside the MASCLI container.

### 2.4 LVM Storage

9. Install LVM operator for OpenShift
```bash
./install_lvm_storage.sh
```

### 2.5 Registry (LVM)

10. Install registry server.
```bash
./install_lvm_registry.sh
```

## 3 Maximo 7.x customization

### 3.1 Instructor-led.

## 4 Maximo 7.x database migration

### 4.1 Instructor-led.

:::info

Your OpenShift must have been installed by now. Jump back up to the Day 2 operations.

:::

## 5 Maximo Application Suite & Manage

```bash
./install_mas.sh
```