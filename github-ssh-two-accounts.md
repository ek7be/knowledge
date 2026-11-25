# GitHub SSH Setup for Two Accounts (Linux)

This guide configures two independent GitHub accounts on one Linux
machine:

-   **Personal** account
-   **Work** account

Each account gets its own Ed25519 SSH key, passphrase, SSH alias, and
Git commit identity.

The setup keeps authentication separate from commit identity, so you can
use the correct GitHub account and author information for each
repository.

## 1. Check your existing SSH directory

``` bash
ls -la ~/.ssh
```

Do not overwrite an existing key unless you are certain you no longer
need it. The commands below use dedicated filenames so they do not
replace the default `id_ed25519` key.

If `~/.ssh` does not exist, the `ssh-keygen` commands in the next
section will create it.

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

You can leave the passphrase empty, but using one provides additional
protection if the private-key file is copied or exposed.

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

Make sure each public key is added to the correct account. GitHub does
not allow the same SSH authentication key to be added to multiple
personal accounts, so keep the two key pairs separate.

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
    AddKeysToAgent yes

# Work GitHub account
Host github-work
    HostName github.com
    User git
    IdentityFile ~/.ssh/id_ed25519_github_work
    IdentitiesOnly yes
    AddKeysToAgent yes
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

`AddKeysToAgent yes` asks SSH to add the key to an available agent when
the key is first used. It does not start an agent or guarantee that the
agent will persist between login sessions; that depends on your Linux
desktop/session configuration.

## 5. Load the keys into an SSH agent

If an SSH agent is already running in your desktop session, you usually
do not need to start another one. Check whether the agent environment
variable is set:

``` bash
echo "$SSH_AUTH_SOCK"
```

A non-empty socket path suggests that an agent is configured for the
shell, but it does not by itself prove that the agent is responding.

Try listing the identities:

``` bash
ssh-add -l
```

Interpret the result:

-   A list of fingerprints means the agent is responding and has
    identities loaded.
-   `The agent has no identities` means the agent is responding but no
    keys are loaded.
-   `Could not open a connection to your authentication agent` means the
    current shell cannot communicate with an agent.

If the agent is available, add the two keys:

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

**Important:** `ssh-add -D` removes every identity from the current
agent, not just these GitHub keys. Other SSH connections that rely on
keys in that agent may be affected.

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
accepting it:

<https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/githubs-ssh-key-fingerprints>

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

Change it to use the appropriate alias if needed.

For a personal repository:

``` bash
git remote set-url origin git@github-personal:PERSONAL_USERNAME/REPOSITORY.git
```

For a work repository:

``` bash
git remote set-url origin git@github-work:ORGANIZATION/REPOSITORY.git
```

**Important:** changing a repository's remote changes how Git connects
to GitHub. It does not change the name or email recorded in new commits.
Configure the commit identity separately as described in Section 8.

Also remember that successful SSH authentication does not automatically
grant access to every repository. The authenticated GitHub account must
have permission to access the repository.

## 8. Set the Git commit identity per repository

SSH keys control **authentication** (which GitHub account you connect
as). Git's `user.name` and `user.email` control the author information
recorded in commits. Configure these separately in each repository.

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

To see where the configured values come from, use:

``` bash
git config --show-origin --get-regexp 'user\.(name|email)'
```

These commands set repository-local values, which take precedence over
global Git identity settings.

The email used for commits should be an email address verified on the
corresponding GitHub account, or an appropriate GitHub-provided
`noreply` address if you prefer to keep your email private.

## 9. Optional: Automatically select the Git identity by directory

If you keep personal and work repositories in separate directories, Git
can select the appropriate commit identity automatically. This reduces
the chance of making a work commit with your personal email, or vice
versa.

For example, organize your repositories like this:

``` text
~/Projects/
    Personal/
        project-one/
        project-two/
    Work/
        company-project-one/
        company-project-two/
```

Edit your global `~/.gitconfig` file and add:

``` ini
[includeIf "gitdir:~/Projects/Personal/"]
    path = ~/.gitconfig-personal

[includeIf "gitdir:~/Projects/Work/"]
    path = ~/.gitconfig-work
```

Create `~/.gitconfig-personal`:

``` ini
[user]
    name = Your Name
    email = personal@example.com
```

Create `~/.gitconfig-work`:

``` ini
[user]
    name = Your Name
    email = work@company.com
```

The directory patterns above match repositories stored inside the
respective directories, including their subdirectories. Adjust the paths
if you use a different directory layout. Matching is case-sensitive on
typical Linux filesystems.

**Precedence note:** repository-local settings (`git config --local`)
take precedence over values supplied by these included files. If a
repository has an old local `user.email` value, that value will continue
to override the conditional identity. Check the effective configuration
with:

``` bash
git config --show-origin --get-regexp 'user\.(name|email)'
```

This section is optional. Setting the identity manually in each
repository, as described in Section 8, is also perfectly valid.

## 10. Quick verification checklist

-   [ ] Two different Ed25519 key pairs exist.
-   [ ] Each private key has its own strong passphrase, if passphrases
    are being used.
-   [ ] Only each `.pub` file was added to its matching GitHub account.
-   [ ] `~/.ssh/config` contains both host aliases.
-   [ ] Private-key and SSH config permissions are restricted.
-   [ ] Both `ssh -T` tests greet the expected GitHub usernames.
-   [ ] Each repository's remote uses the correct alias.
-   [ ] Each repository has the correct Git commit name and email.
-   [ ] The effective Git identity has been checked with
    `git config --show-origin`.

## 11. Troubleshooting

### SSH authenticates as the wrong GitHub account

Run SSH in verbose mode:

``` bash
ssh -vT git@github-personal
```

Or:

``` bash
ssh -vT git@github-work
```

Look for the `Offering public key` messages. They show which key SSH is
attempting to use. Confirm that the relevant alias in `~/.ssh/config`
points to the intended private key and includes `IdentitiesOnly yes`.

### Permission denied (publickey)

Check the agent:

``` bash
ssh-add -l
```

Then verify that:

-   The correct public key is registered with the correct GitHub
    account.
-   The corresponding private key exists at the path specified in
    `~/.ssh/config`.
-   The SSH alias is spelled correctly.
-   The private key is readable by your user and has restrictive
    permissions.
-   You are connecting with the intended alias, such as
    `github-personal` or `github-work`.

You can also inspect the effective SSH configuration for an alias:

``` bash
ssh -G github-personal
```

### Commits appear under the wrong identity

Check the effective Git identity and where it was configured:

``` bash
git config --show-origin --get-regexp 'user\.(name|email)'
```

Set the correct identity in the repository if necessary:

``` bash
git config user.name "Your Name"
git config user.email "correct@example.com"
```

If you use the optional conditional configuration in Section 9, check
whether an existing repository-local value is overriding it.

### Repository access is denied

Successful SSH authentication confirms which GitHub account you
connected as; it does not prove that the account has access to the
requested repository.

Check that:

-   The repository owner and name in the remote URL are correct.
-   The authenticated account has permission to access the repository.
-   For an organization repository, any required organization access or
    SSO authorization has been completed.

## 12. Ongoing maintenance

You do not need to rotate healthy SSH keys on an arbitrary schedule.
Review the keys registered in both GitHub accounts periodically and
remove keys for devices you no longer use.

Replace and revoke a key promptly if its private part may have been
exposed, a device containing it is lost or stolen, or the device is
decommissioned. Use a separate key per device rather than copying one
private key across machines.

Keep GitHub two-factor authentication enabled as an additional layer of
account protection.

GitHub SSH key settings:

<https://github.com/settings/keys>
