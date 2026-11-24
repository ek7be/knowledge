# GitHub SSH Setup for Two Accounts (Linux)

This guide configures two independent GitHub accounts on one Linux
machine:

-   **Personal** account
-   **Work** account

Each account gets its own Ed25519 SSH key, passphrase, SSH alias, and
Git commit identity.

## 1. Check your existing SSH directory

``` bash
ls -la ~/.ssh
```

Do not overwrite an existing key unless you are certain you no longer
need it. The commands below use dedicated filenames so they do not
replace the default `id_ed25519` key.

## 2. Generate one key for each account

### Personal account

``` bash
ssh-keygen -t ed25519 \
  -C "personal@example.com" \
  -f ~/.ssh/id_ed25519_github_personal
```

### Work account

``` bash
ssh-keygen -t ed25519 \
  -C "work@company.com" \
  -f ~/.ssh/id_ed25519_github_work
```

Replace the example email addresses with the email addresses associated
with the respective GitHub accounts.

When prompted for a passphrase, use a **strong, unique passphrase for
each key**. Do not reuse your GitHub account password. A password
manager is a good place to store both passphrases.

Each command creates two files:

  -----------------------------------------------------------------------
  File                                Purpose
  ----------------------------------- -----------------------------------
  `id_ed25519_github_personal` /      Private key --- keep secret
  `id_ed25519_github_work`            

  Corresponding `.pub` file           Public key --- this is what you add
                                      to GitHub
  -----------------------------------------------------------------------

Never send, upload, or commit a private key. Only the `.pub` public key
is shared with GitHub.

## 3. Add each public key to the matching GitHub account

Print the personal public key:

``` bash
cat ~/.ssh/id_ed25519_github_personal.pub
```

Sign in to your **personal** GitHub account and add that public key at:

<https://github.com/settings/ssh/new>

Use a recognizable title, such as `Linux laptop - personal`, and select
**Authentication Key**.

Then print the work public key:

``` bash
cat ~/.ssh/id_ed25519_github_work.pub
```

Sign in to your **work** GitHub account and add that public key in the
same place. Use a title such as `Linux laptop - work`.

Make sure each public key is added to the correct account.

## 4. Configure SSH aliases

Create or edit `~/.ssh/config`:

``` bash
nano ~/.ssh/config
```

Add:

``` sshconfig
# Personal GitHub account
Host github-personal
    HostName github.com
    User git
    IdentityFile ~/.ssh/id_ed25519_github_personal
    IdentitiesOnly yes

# Work GitHub account
Host github-work
    HostName github.com
    User git
    IdentityFile ~/.ssh/id_ed25519_github_work
    IdentitiesOnly yes
```

Save the file, then set permissions:

``` bash
chmod 700 ~/.ssh
chmod 600 ~/.ssh/config
chmod 600 ~/.ssh/id_ed25519_github_personal
chmod 600 ~/.ssh/id_ed25519_github_work
chmod 644 ~/.ssh/id_ed25519_github_personal.pub
chmod 644 ~/.ssh/id_ed25519_github_work.pub
```

## 5. Load the keys into an SSH agent

If an SSH agent is already running in your desktop session, you usually
do not need to start another one. Check whether the agent environment
variable is set:

``` bash
echo "$SSH_AUTH_SOCK"
```

If that prints a socket path, try adding the keys:

``` bash
ssh-add ~/.ssh/id_ed25519_github_personal
ssh-add ~/.ssh/id_ed25519_github_work
```

Enter each key's passphrase when prompted.

If no agent is available in your current shell, you can start one for
that shell session:

``` bash
eval "$(ssh-agent -s)"
```

Then run the two `ssh-add` commands above.

List keys currently loaded in the agent:

``` bash
ssh-add -l
```

Remove all identities from the current agent:

``` bash
ssh-add -D
```

**Agent behavior:** an agent caches unlocked keys while it is running.
Whether the keys remain available after logging out or restarting
depends on your desktop environment and agent configuration. Avoid
starting multiple agents unnecessarily; use the agent already provided
by your Linux session when possible.

## 6. Test both GitHub accounts

Test the personal alias:

``` bash
ssh -T git@github-personal
```

Test the work alias:

``` bash
ssh -T git@github-work
```

The first connection may ask you to confirm GitHub's host key. Verify
the fingerprint against GitHub's published SSH key fingerprints before
accepting it.

A successful authentication response looks similar to:

``` text
Hi USERNAME! You've successfully authenticated, but GitHub does not provide shell access.
```

Check that the username in each response is the expected account.

## 7. Clone repositories using the matching alias

### Personal repository

``` bash
git clone git@github-personal:PERSONAL_USERNAME/REPOSITORY.git
```

### Work repository

``` bash
git clone git@github-work:ORGANIZATION/REPOSITORY.git
```

Replace the placeholders with the actual owner and repository names.

For an existing repository, inspect its remote:

``` bash
git remote -v
```

Change it to use the appropriate alias if needed:

``` bash
git remote set-url origin git@github-personal:PERSONAL_USERNAME/REPOSITORY.git
```

Or, for a work repository:

``` bash
git remote set-url origin git@github-work:ORGANIZATION/REPOSITORY.git
```

## 8. Set the Git commit identity per repository

SSH keys control **authentication** (which GitHub account you connect
as). Git's `user.name` and `user.email` control the **author information
recorded in commits**. Configure these separately in each repository.

Inside a personal repository:

``` bash
git config user.name "Your Name"
git config user.email "personal@example.com"
```

Inside a work repository:

``` bash
git config user.name "Your Name"
git config user.email "work@company.com"
```

Check the identity configured for the current repository:

``` bash
git config user.name
git config user.email
```

These commands set repository-local values, which take precedence over
global Git identity settings.

## 9. Quick verification checklist

-   [ ] Two different Ed25519 key pairs exist.
-   [ ] Each private key has its own strong passphrase.
-   [ ] Only each `.pub` file was added to its matching GitHub account.
-   [ ] `~/.ssh/config` contains both host aliases.
-   [ ] Private-key and SSH config permissions are restricted.
-   [ ] Both `ssh -T` tests greet the expected GitHub usernames.
-   [ ] Each repository's remote uses the correct alias.
-   [ ] Each repository has the correct Git commit name and email.

## 10. Ongoing maintenance

You do not need to rotate healthy SSH keys on an arbitrary schedule.
Review the keys registered in both GitHub accounts periodically and
remove keys for devices you no longer use.

Replace and revoke a key promptly if its private part may have been
exposed, a device containing it is lost or stolen, or the device is
decommissioned. Use a separate key per device rather than copying one
private key across machines.

Keep GitHub two-factor authentication enabled as an additional layer of
account protection.

GitHub SSH key settings: <https://github.com/settings/keys>
