# Jenkins SSH to a VM: system account or Jenkins Credentials

You do not need the Jenkins SSH Agent plugin\. Use the Linux `ssh` command with either a dedicated system account or a private key supplied by Jenkins Credentials\.

## Understand the accounts

|Term                 |Meaning                                                                                                              |
|---------------------|---------------------------------------------------------------------------------------------------------------------|
|Jenkins build agent  |Machine or container executing your pipeline commands. It can also be the controller if configured to run builds.    |
|SSH Agent plugin     |Optional Jenkins plugin that loads credentials into an SSH authentication agent. Neither option below requires it.   |
|Local system account |Linux account on the machine executing the job; for example, `jenkins-ssh`. It can own the SSH key and configuration.|
|Remote system account|Linux account on your target VM; for example, `ci-runner`. Its authorized keys determine who can log in.             |

A system account can own and perform the SSH operation\. Jenkins must still start that operation somewhere\. If your goal is to run it on the controller, select that machine’s build label; configuring a system account alone does not move execution there\.

These instructions assume Linux on both ends\. Example values are placeholders:

|Setting                                        |Example      |
|-----------------------------------------------|-------------|
|Local account that executes Jenkins shell steps|`jenkins`    |
|Dedicated local SSH account                    |`jenkins-ssh`|
|Remote VM account                              |`ci-runner`  |
|VM address                                     |`10.0.0.139` |
|Label of the machine configured for SSH        |`ssh-runner` |
|SSH alias                                      |`my-vm`      |

## Choose an option

|Option                           |Key location                                                             |Jenkins requirement                                                       |Best fit                                                                                  |
|---------------------------------|-------------------------------------------------------------------------|--------------------------------------------------------------------------|------------------------------------------------------------------------------------------|
|1. Dedicated local system account|`/var/lib/jenkins-ssh/.ssh/` on the executing machine                    |Standard pipeline shell step plus permission to invoke a restricted helper|You want a system account to own the SSH configuration, without extra Jenkins SSH plugins.|
|2. Jenkins Credentials           |Jenkins-managed credential, temporarily supplied to the executing machine|Credentials Binding and SSH Credentials support                           |You want to manage the private key through Jenkins.                                       |

Recommendation for your preference: **Option 1**\. Set up the system account once, provide Jenkins a small helper, and call that helper from the pipeline\.

## Shared setup: create the remote VM account

An administrator runs these commands on the target VM &#40;Debian/Ubuntu\-style tools&#41;:

```bash
sudo useradd --system --create-home \
    --home-dir /var/lib/ci-runner \
    --shell /bin/bash ci-runner

sudo install -d -o ci-runner -g ci-runner -m 700 \
    /var/lib/ci-runner/.ssh
```

If using an existing VM login account, skip account creation and use that account and its actual home directory throughout\.

The VM needs a running SSH server, reachable TCP port 22, and public\-key authentication enabled\. Some distributions reject public\-key login to locked accounts, or restrict system accounts through PAM or `AllowUsers`\. Have the administrator configure this account to permit key\-only SSH login under your site’s policy\. Giving it `/bin/bash` enables remote shell commands; `/usr/sbin/nologin` would prevent this workflow\.

The remote account also needs access to the commands, scripts, and directories your pipeline will use\. Being a system account does not automatically grant administrative rights\.

## Option 1: a dedicated local system account

### 1\. Create the local account

On the machine that will execute Jenkins shell steps, an administrator runs:

```bash
sudo useradd --system --create-home \
    --home-dir /var/lib/jenkins-ssh \
    --shell /bin/bash jenkins-ssh

sudo install -d -o jenkins-ssh -g jenkins-ssh -m 700 \
    /var/lib/jenkins-ssh/.ssh
```

If this account exists, reuse its actual home directory instead of recreating it\.

### 2\. Generate its dedicated key

```bash
sudo -u jenkins-ssh -H ssh-keygen \
    -t ed25519 \
    -f /var/lib/jenkins-ssh/.ssh/id_ed25519 \
    -N ""
```

Do not overwrite an existing key\. An empty passphrase permits unattended operation without an SSH authentication agent\. If Ed25519 is disallowed by your environment, use an approved key type instead\.

Display only the public key:

```bash
sudo cat /var/lib/jenkins-ssh/.ssh/id_ed25519.pub
```

### 3\. Install that public key on the VM

On the VM, append the complete public\-key line:

```bash
sudo tee -a /var/lib/ci-runner/.ssh/authorized_keys >/dev/null <<'PUBLIC_KEY'
PASTE_THE_COMPLETE_PUBLIC_KEY_LINE_HERE
PUBLIC_KEY

sudo chown ci-runner:ci-runner \
    /var/lib/ci-runner/.ssh/authorized_keys
sudo chmod 600 /var/lib/ci-runner/.ssh/authorized_keys
```

Replace the placeholder before running\. Preserve existing authorized keys\.

### 4\. Configure the local account’s SSH alias

On the executing machine, add the following entry to `/var/lib/jenkins-ssh/.ssh/config`, preserving any existing entries:

```sshconfig
Host my-vm
    HostName 10.0.0.139
    User ci-runner
    Port 22
    IdentityFile /var/lib/jenkins-ssh/.ssh/id_ed25519
    IdentitiesOnly yes
    BatchMode yes
    ConnectTimeout 10
    StrictHostKeyChecking yes
```

Set ownership and permissions:

```bash
sudo chown jenkins-ssh:jenkins-ssh \
    /var/lib/jenkins-ssh/.ssh/config
sudo chmod 600 /var/lib/jenkins-ssh/.ssh/config
```

### 5\. Verify and register the VM host key

From the VM console or another trusted connection, obtain its host\-key fingerprint:

```bash
sudo ssh-keygen -lf /etc/ssh/ssh_host_ed25519_key.pub
```

On the executing machine, connect interactively once as the dedicated account:

```bash
sudo -u jenkins-ssh -H ssh \
    -o StrictHostKeyChecking=ask my-vm 'hostname; whoami'
```

Compare the displayed fingerprint before accepting\. If a different algorithm is presented, compare its corresponding host public\-key file on the VM\.

Confirm unattended login:

```bash
sudo -u jenkins-ssh -H ssh my-vm 'hostname; whoami'
```

### 6\. Give Jenkins a restricted helper

Create `/usr/local/bin/jenkins-vm-check` on the executing machine with this content:

```sh
#!/bin/sh
set -eu
if [ "$#" -ne 0 ]; then
    echo "This helper accepts no arguments" >&2
    exit 2
fi
exec /usr/bin/ssh -F /var/lib/jenkins-ssh/.ssh/config \
    my-vm 'hostname; whoami'
```

Confirm the actual SSH binary path with `command -v ssh`; adjust the helper if needed\.

Make the helper administrator\-owned so Jenkins cannot change it:

```bash
sudo chown root:root /usr/local/bin/jenkins-vm-check
sudo chmod 755 /usr/local/bin/jenkins-vm-check
```

Edit a sudoers file with validation:

```bash
sudo visudo -f /etc/sudoers.d/jenkins-vm
```

Add this rule, replacing `jenkins` if your pipeline uses another OS account:

```sudoers
jenkins ALL=(jenkins-ssh) NOPASSWD: /usr/local/bin/jenkins-vm-check ""
```

The `""` argument specification permits this helper only without arguments\. The helper also rejects arguments\. This avoids granting Jenkins unrestricted `sudo` access to the dedicated account\.

Test as the Jenkins OS account:

```bash
sudo -n -u jenkins-ssh -H /usr/local/bin/jenkins-vm-check
```

For a real task, create another administrator\-owned helper using the same pattern, with a fixed remote command such as:

```sh
exec /usr/bin/ssh -F /var/lib/jenkins-ssh/.ssh/config \
    my-vm 'python3 /opt/scripts/my_script.py'
```

Give that helper its own exact sudoers entry\. Remote scripts must exist and be readable by `ci-runner`\.

### 7\. Jenkinsfile

```groovy
pipeline {
    agent { label 'ssh-runner' }

    options {
        timestamps()
        timeout(time: 10, unit: 'MINUTES')
    }

    stages {
        stage('Check VM using system account') {
            steps {
                sh 'sudo -n -u jenkins-ssh -H /usr/local/bin/jenkins-vm-check'
            }
        }
    }
}
```

The Jenkins job starts the helper, but the SSH process runs as `jenkins-ssh` and uses that account’s configuration and key\. No SSH Agent plugin or Credentials Binding step is needed\.

## Option 2: Jenkins\-managed SSH credentials

This option requires Credentials Binding and SSH Credentials support, but no SSH Agent plugin\. SSH runs as the normal pipeline OS account and receives a temporary key file from Jenkins\.

### 1\. Generate a dedicated key

On a trusted administrator machine:

```bash
ssh-keygen -t ed25519 -f ./jenkins_vm_key -N ""
```

Use a fresh filename\. Install `jenkins_vm_key.pub` in the VM account’s `authorized_keys` using the shared setup and append procedure above\.

### 2\. Add the credential in Jenkins

Add a credential accessible to the job:

|Field      |Value                        |
|-----------|-----------------------------|
|Kind       |SSH Username with private key|
|Username   |`ci-runner`                  |
|Private key|Contents of `jenkins_vm_key` |
|Passphrase |Empty for this example       |
|ID         |`vm-ssh-key`                 |

Keep the private key out of Git\. A passphrase is not automatically handled by `ssh -i`; this minimal example uses a key without one\.

### 3\. Provision verified host trust

The normal OS account executing Jenkins shell steps needs the VM’s verified host key in its own `~/.ssh/known_hosts`\.

After checking the fingerprint through the trusted VM console, an administrator can register it as that account:

```bash
sudo -u jenkins -H bash
mkdir -p ~/.ssh
chmod 700 ~/.ssh
ssh -o StrictHostKeyChecking=ask -o PreferredAuthentications=none \
    ci-runner@10.0.0.139
```

Accept only the matching fingerprint\. Authentication failure is expected here; this command records host trust without supplying a login key\. If Jenkins runs under another account, replace `jenkins`\.

### 4\. Jenkinsfile

```groovy
pipeline {
    agent { label 'ssh-runner' }

    options {
        timestamps()
        timeout(time: 10, unit: 'MINUTES')
    }

    stages {
        stage('Check VM using Jenkins Credentials') {
            steps {
                withCredentials([sshUserPrivateKey(
                    credentialsId: 'vm-ssh-key',
                    keyFileVariable: 'SSH_KEY',
                    usernameVariable: 'SSH_USER'
                )]) {
                    sh '''
                        ssh -i "$SSH_KEY" \
                            -o IdentitiesOnly=yes \
                            -o BatchMode=yes \
                            -o StrictHostKeyChecking=yes \
                            -o ConnectTimeout=10 \
                            "$SSH_USER@10.0.0.139" 'hostname; whoami'
                    '''
                }
            }
        }
    }
}
```

Use triple\-single\-quoted Groovy strings so the shell expands credential variables\. Replace the remote command with the script or operation you need\. A nonzero remote exit status normally fails the Jenkins shell step\.

## If the Jenkins service already runs as a system account

If your intent is simply to use the existing `jenkins` system account, you can put the key, config, and verified known\-hosts file in that account’s home directory\. You do not need the second local account or sudo helper\.

After configuring an alias as in Option 1, with paths adjusted to that account’s home, the pipeline can use:

```groovy
sh "ssh my-vm 'hostname; whoami'"
```

This is the shortest setup, but all jobs running under that account can potentially use the key\.

## Controller, agents, and Docker

|Execution environment              |Where to configure SSH                                                                                  |
|-----------------------------------|--------------------------------------------------------------------------------------------------------|
|Persistent Linux build machine     |On that machine, under the account selected by your option.                                             |
|Jenkins controller executing builds|On the controller; target its actual build label. It must be enabled to execute jobs.                   |
|Docker build container             |Inside the container’s execution environment; host accounts and helpers are not automatically available.|
|Temporary build machine            |Provision required accounts/configuration on creation; Jenkins Credentials can provide the private key. |

Use a label targeting the configured environment\. `agent any` can select a machine without your SSH setup\. Never bake a private key into a container image\.

## Troubleshooting

|Problem                                         |What to check                                                                                               |
|------------------------------------------------|------------------------------------------------------------------------------------------------------------|
|`ssh: not found`                                |SSH client and binary path on the executing machine/container.                                              |
|`sudo: a password is required`                  |Actual pipeline OS account, exact helper path, run-as account, and matching sudoers rule.                   |
|`Permission denied (publickey)`                 |VM username, installed public key, private key, ownership, account lock/PAM policy, and SSH server settings.|
|`Host key verification failed`                  |Verified `known_hosts` file belonging to the account actually running SSH.                                  |
|`REMOTE HOST IDENTIFICATION HAS CHANGED`        |Verify the VM identity and fingerprint before replacing its old entry.                                      |
|Connection timeout                              |Routing, firewall, VM address, and port from the executing machine.                                         |
|Connection refused                              |VM SSH server and listening port.                                                                           |
|Works manually but fails in Jenkins             |Compare machine/container, OS account, home directory, and permissions.                                     |
|Missing `withCredentials` or `sshUserPrivateKey`|Required credential support is unavailable; use Option 1 or install that support.                           |
|Remote script cannot write files                |Permissions for the remote `ci-runner` account.                                                             |

Diagnose Option 1 as its account:

```bash
sudo -u jenkins-ssh -H ssh -vvv my-vm 'hostname'
```

SSH logs can identify which key was offered and why authentication failed\. Do not paste private keys into logs\.

## References

- [Jenkins SSH Agent documentation: direct SSH alternative](https://plugins.jenkins.io/ssh-agent/)
- [Jenkins Credentials Binding reference](https://www.jenkins.io/doc/pipeline/steps/credentials-binding/)

All comparison tables in this README use standard Markdown\. No Mermaid renderer is required\.
