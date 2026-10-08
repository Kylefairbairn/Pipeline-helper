SSH from a dev VM to a boot server using a system account

Configure the boot server’s dedicated account bootci for SSH key login from your dev VM. Generate the private key on the dev VM and install only its public key on the boot server. No Jenkins SSH Agent plugin is required.

This guide uses standard Markdown tables and code blocks; it does not require Mermaid.

Machines and example values

|Machine    |Role                               |What it stores                               |
|-----------|-----------------------------------|---------------------------------------------|
|Dev VM     |Starts SSH connections             |Private key and verified boot-server host key|
|Boot server|Accepts SSH connections as `bootci`|Public key in `bootci`’s authorized_keys     |

|Setting                    |Example                |
|---------------------------|-----------------------|
|Boot server IP             |`10.0.0.139`           |
|Boot server system account |`bootci`               |
|Boot server account home   |`/var/lib/bootci`      |
|Dev VM private-key filename|`~/.ssh/bootserver_key`|
|SSH port                   |`22`                   |

Replace the example IP if different. Run dev VM commands as the local user that will initiate SSH. A second local system account is unnecessary. If Jenkins eventually executes SSH, that execution account needs access to the private key and verified host trust.

1. Prepare the system account on the BOOT SERVER

These commands use Linux useradd and require administrator access.

For a new account:

sudo useradd --system --create-home \
    --home-dir /var/lib/bootci \
    --shell /bin/bash bootci

For an existing account, inspect it instead:

getent passwd bootci
sudo passwd -S bootci
sudo chage -l bootci

Use its actual home directory throughout if different from /var/lib/bootci.

The account needs a usable shell to execute remote commands:

sudo usermod -s /bin/bash bootci

Do not set its shell to nologin or false for this workflow.

Account lock versus key-only authentication

Do not use passwd -l or usermod -L to enforce key-only SSH. Depending on the distribution and PAM configuration, a locked account can reject public-key authentication too.

A straightforward way to give the account a valid password state is:

sudo passwd bootci

Set a strong password interactively. You will disable SSH password authentication for this account below; the password is not put in the SSH command or pipeline. Existing account expiry or PAM restrictions must also permit login. This does not disable password use for other services or the local console.

Create the public-key directory:

sudo install -d -o bootci -g "$(id -gn bootci)" -m 700 \
    /var/lib/bootci/.ssh

2. Choose a key type on the DEV VM

Because you previously had a key-type problem, start with RSA 3072 unless your environment requires another choice. RSA is a practical compatibility option; both machines’ cryptographic policies still have to allow it.

|Key type            |Use when                                                            |Notes                                                        |
|--------------------|--------------------------------------------------------------------|-------------------------------------------------------------|
|RSA 3072 or 4096    |Compatibility is your main concern, including many FIPS environments|Modern SSH uses RSA with SHA-2 signatures.                   |
|Ed25519             |Both machines are modern and policy permits it                      |Compact; unavailable in RHEL OpenSSH FIPS mode.              |
|ECDSA P-256 or P-384|Your environment permits NIST curve keys                            |Another option for environments where Ed25519 is unavailable.|

Check the client version and supported key algorithms:

ssh -V
ssh -Q key

Supported algorithms are not necessarily permitted by the active policy. On systems exposing the Linux FIPS flag:

cat /proc/sys/crypto/fips_enabled

1 indicates FIPS mode; a missing file does not establish the absence of policy restrictions. On RHEL-family systems with the tool installed:

update-crypto-policies --show

Check both machines when diagnosing policy incompatibility.

3. Generate ONE key on the DEV VM

Prepare the directory:

mkdir -p ~/.ssh
chmod 700 ~/.ssh

Choose one command below. They intentionally use the same filename so the remaining instructions are identical. Do not overwrite an existing key; choose another filename if necessary.

Option A: RSA 3072 (start here for compatibility)

ssh-keygen -t rsa -b 3072 \
    -f ~/.ssh/bootserver_key \
    -C 'dev-vm-to-bootserver' -N ''

Option B: RSA 4096

ssh-keygen -t rsa -b 4096 \
    -f ~/.ssh/bootserver_key \
    -C 'dev-vm-to-bootserver' -N ''

Option C: Ed25519

ssh-keygen -t ed25519 \
    -f ~/.ssh/bootserver_key \
    -C 'dev-vm-to-bootserver' -N ''

Option D: ECDSA P-256

ssh-keygen -t ecdsa -b 256 \
    -f ~/.ssh/bootserver_key \
    -C 'dev-vm-to-bootserver' -N ''

For P-384, use -b 384 instead, if allowed by your policy.

-N '' creates a key without a passphrase for unattended use. For manual use with a passphrase, omit -N ''; SSH will prompt for it when necessary.

Two files are created:

|File                       |Purpose                              |
|---------------------------|-------------------------------------|
|`~/.ssh/bootserver_key`    |Private key; remains on the dev VM   |
|`~/.ssh/bootserver_key.pub`|Public key; copied to the boot server|

chmod 600 ~/.ssh/bootserver_key
ssh-keygen -lf ~/.ssh/bootserver_key.pub

Optional RSA PEM format for an older consuming tool

If a tool specifically cannot read the modern OpenSSH private-key format, generate an RSA key in PEM format using a different filename:

ssh-keygen -t rsa -b 3072 -m PEM \
    -f ~/.ssh/bootserver_key_pem \
    -C 'dev-vm-to-bootserver' -N ''

Use that filename for subsequent commands. PEM changes the private-key encoding, not server authentication policy. It will not fix a locked account or an unauthorized public key. Ordinary modern OpenSSH does not need this option.

4. Install the public key on the BOOT SERVER

Choose one installation method.

Method A: ssh-copy-id

If the account currently permits password SSH login:

ssh-copy-id -i ~/.ssh/bootserver_key.pub bootci@10.0.0.139

Verify the server fingerprint before accepting the first connection, and enter the account password. This may fail if password login is already disabled; use Method B instead.

Method B: copy through the boot server console or admin login

On the DEV VM, display the public key:

cat ~/.ssh/bootserver_key.pub

On the BOOT SERVER, append that complete single line:

sudo tee -a /var/lib/bootci/.ssh/authorized_keys >/dev/null <<'PUBLIC_KEY'
PASTE_THE_COMPLETE_PUBLIC_KEY_LINE_HERE
PUBLIC_KEY

Replace the placeholder before running. Do not paste the private key. Preserve existing authorized keys.

On the BOOT SERVER, set permissions:

sudo chown bootci:"$(id -gn bootci)" \
    /var/lib/bootci/.ssh/authorized_keys
sudo chmod 600 /var/lib/bootci/.ssh/authorized_keys
sudo chmod go-w /var/lib/bootci

If SELinux is enabled and restorecon is available:

sudo restorecon -Rv /var/lib/bootci

A nonstandard home path may need administrator-defined SELinux file-context mappings; consult audit logs if access is still denied.

Compare the installed key fingerprint on the BOOT SERVER:

sudo ssh-keygen -lf /var/lib/bootci/.ssh/authorized_keys

It should include the fingerprint from the dev VM’s public key.

5. Require key authentication on the BOOT SERVER

Edit /etc/ssh/sshd_config as an administrator. Add this block at the end, taking any existing Match blocks into account:

Match User bootci
    PubkeyAuthentication yes
    AuthenticationMethods publickey
    PasswordAuthentication no
    KbdInteractiveAuthentication no

For older OpenSSH releases that reject KbdInteractiveAuthentication, consult the installed man sshd_config; the older spelling is ChallengeResponseAuthentication. Validate the version-specific configuration before applying it.

Keep an existing administrator session open. Check syntax:

sudo sshd -t

If sshd is not in sudo’s command search path, use its actual path, commonly /usr/sbin/sshd. Do not reload if validation fails.

Reload the service matching your distribution; run only the applicable command:

sudo systemctl reload ssh

Or:

sudo systemctl reload sshd

Additional AllowUsers, DenyUsers, PAM, expiry, and cryptographic policies can still affect the account. An administrator can inspect the effective settings for the real dev VM source address:

sudo sshd -T -C user=bootci,addr=DEV_VM_IP,host=DEV_VM_HOSTNAME

Replace the connection placeholders. Look at pubkeyauthentication, authenticationmethods, authorizedkeysfile, and password settings.

6. Verify the boot server identity and test from the DEV VM

On the BOOT SERVER through its trusted console/admin session:

sudo ssh-keygen -lf /etc/ssh/ssh_host_ed25519_key.pub

If the connection presents an RSA or ECDSA host key, inspect /etc/ssh/ssh_host_rsa_key.pub or /etc/ssh/ssh_host_ecdsa_key.pub instead. These are server host keys, separate from your login key.

From the DEV VM:

ssh -i ~/.ssh/bootserver_key \
    -o IdentitiesOnly=yes \
    -o PreferredAuthentications=publickey \
    bootci@10.0.0.139 'whoami; hostname'

On the first connection, compare the fingerprint before accepting it. Success prints bootci and the boot server’s hostname.

Then verify unattended operation:

ssh -i ~/.ssh/bootserver_key \
    -o IdentitiesOnly=yes \
    -o BatchMode=yes \
    -o StrictHostKeyChecking=yes \
    bootci@10.0.0.139 'whoami'

7. Simplify to an SSH alias on the DEV VM

Add this entry to ~/.ssh/config, preserving existing entries:

Host bootserver
    HostName 10.0.0.139
    User bootci
    Port 22
    IdentityFile ~/.ssh/bootserver_key
    IdentitiesOnly yes
    BatchMode yes
    StrictHostKeyChecking yes
    ConnectTimeout 10

chmod 600 ~/.ssh/config
ssh bootserver 'whoami; hostname'

Run your actual boot-server script:

ssh bootserver 'python3 /opt/scripts/netboot.py'

Replace the script path. The account must have permission to execute the operation; system accounts do not automatically have sudo rights.

8. Jenkins later, without the SSH Agent plugin

If Jenkins executes on the dev VM as the same configured OS account:

sh "ssh bootserver 'whoami; hostname'"

If Jenkins runs as another user or inside a container, your successful interactive login does not configure that environment. Provision its own dedicated key and verified host trust, or supply a key through Jenkins Credentials Binding if available. The public key belongs in the same boot-server account’s authorized_keys.

Troubleshooting permission denied

On the DEV VM:

ssh -vvv -i ~/.ssh/bootserver_key \
    -o IdentitiesOnly=yes \
    -o PreferredAuthentications=publickey \
    bootci@10.0.0.139 'whoami'

On the BOOT SERVER immediately afterward:

sudo journalctl -u ssh -u sshd --since '5 minutes ago' --no-pager
sudo passwd -S bootci
getent passwd bootci
sudo chage -l bootci

Depending on the distribution, authentication messages may instead be in /var/log/auth.log or /var/log/secure.

|Message or symptom                              |Check or next step                                                                                                                            |
|------------------------------------------------|----------------------------------------------------------------------------------------------------------------------------------------------|
|`Permission denied (publickey)`                 |Key installation, username, account status, server logs, permissions, and policy. This alone does not identify the cause.                     |
|`User ... not allowed because account is locked`|Set a valid password state and enforce key-only SSH through sshd configuration; check PAM/expiry too.                                         |
|No `Offering public key` line                   |Correct private-key path, readability, and supported key type.                                                                                |
|`Offering public key` but no acceptance         |Installed key fingerprint, actual account home, AuthorizedKeysFile, permissions, and server policy.                                           |
|`Server accepts key` followed by signing failure|Private-key access, passphrase, crypto policy, or client/provider failure.                                                                    |
|Ed25519 disallowed in FIPS mode                 |Generate and install an allowed RSA or ECDSA key.                                                                                             |
|`no mutual signature algorithm`                 |Compare client/server versions and accepted signature policies; a different allowed key or software update may be needed.                     |
|`invalid format` or `error in libcrypto`        |Check for a wrong file, damaged key/newlines, incompatible encoding, or crypto-policy restriction. PEM helps only an encoding incompatibility.|
|`bad ownership or modes`                        |Fix ownership and write permissions on home, .ssh, and authorized_keys.                                                                       |
|`This account is currently not available`       |Replace nologin/false with a usable shell if remote command execution is intended.                                                            |
|Works as you, fails in Jenkins                  |Different execution account, home, machine, container, or key access.                                                                         |

An RSA public key commonly starts with ssh-rsa. That does not mean modern SSH must use the obsolete SHA-1 ssh-rsa signature algorithm: RSA keys also work with RSA-SHA2 signatures. Do not enable SHA-1 globally to work around an unidentified failure.

To reconstruct a missing public-key file from its private key on the DEV VM:

ssh-keygen -y -f ~/.ssh/bootserver_key > ~/.ssh/bootserver_key.pub

To inspect the actual client configuration:

ssh -G bootserver

Sources

• OpenSSH ssh-keygen manual
• OpenSSH server account accessibility and authorized keys
• Red Hat OpenSSH configuration and Ed25519/FIPS restrictions
• OpenSSH release notes, including RSA-SHA2 versus ssh-rsa
