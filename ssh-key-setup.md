Private key = your secret key (stays on your Mac)
Public key = the copy you give to GitHub or servers
SSH proves your identity without using passwords every time.

For your use case (GitHub + Linux servers), you'll typically do this setup once and reuse the same key everywhere.

1. Check if SSH keys already exist

Run:

ls -al ~/.ssh

Explanation
Part	Meaningls	List files
-a	Show hidden files
-l	Show detailed information
~	Your home directory
~/.ssh	SSH configuration folder

Look for:

id_ed25519
id_ed25519.pub


If they don't exist, create them.

2. Generate a new SSH key

Run:

ssh-keygen -t ed25519 -C "your-email@example.com"


Example:

ssh-keygen -t ed25519 -C "reza@example.com"

Explanation
Part	Meaningssh-keygen	SSH key creation tool
-t	Key type
ed25519	Modern, secure SSH algorithm
-C	Comment added to the key (usually email)

Terminal asks:

Enter file in which to save the key:


Press:

Enter


to use default location:

~/.ssh/id_ed25519


Then:

Enter passphrase:


You can:

Option A (Recommended)

Enter a password.

Pros:

More secure
Safe if laptop is stolen
Option B

Press Enter twice.

Pros:

Faster
No extra prompts

Cons:

Slightly less secure

Created files:

~/.ssh/id_ed25519


Private key 🔒

Never share.

and

~/.ssh/id_ed25519.pub


Public key 🔑

Safe to share.

3. Start SSH Agent

Run:

eval "$(ssh-agent -s)"

Explanation
Part	Meaningssh-agent	Background service remembering SSH keys
-s	Output shell commands
eval	Execute those commands

Without the agent, you'd repeatedly enter your passphrase.

4. Add your key to the agent

Modern macOS:

ssh-add --apple-use-keychain ~/.ssh/id_ed25519

Explanation
Part	Meaningssh-add	Register a key with SSH agent
--apple-use-keychain	Save passphrase in macOS Keychain
~/.ssh/id_ed25519	Your private key

Older macOS:

ssh-add ~/.ssh/id_ed25519

5. Create SSH Configuration

Open config file:

nano ~/.ssh/config

Explanation
Part	Meaningnano	Terminal text editor
~/.ssh/config	SSH configuration file

Add:

Host *
    AddKeysToAgent yes
    UseKeychain yes
    IdentityFile ~/.ssh/id_ed25519

Explanation
Host *
Host *


Means:

Apply this configuration to all SSH connections.

The * means "everything".

AddKeysToAgent yes
AddKeysToAgent yes


Means:

Automatically add keys to SSH agent.

Without it:

ssh-add ...


might be needed repeatedly.

UseKeychain yes
UseKeychain yes


Means:

Store passphrase in macOS Keychain.

So you usually enter it once.

IdentityFile
IdentityFile ~/.ssh/id_ed25519


Means:

Always use this private key.

Without this line, SSH may try multiple keys.

Save in Nano:

Ctrl + O
Enter
Ctrl + X

Explanation
Key	MeaningCtrl+O	Save
Enter	Confirm
Ctrl+X	Exit
6. Secure the configuration file

Run:

chmod 600 ~/.ssh/config

Explanation
Part	Meaningchmod	Change permissions
600	Only you can read/write
~/.ssh/config	Config file

SSH likes configuration files to be private.

7. View your public key

Run:

cat ~/.ssh/id_ed25519.pub

Explanation
Part	Meaningcat	Display file contents
id_ed25519.pub	Public key

Output:

ssh-ed25519 AAAA.... reza@example.com


Copy the entire line.

8. Add Key to GitHub

Open:

GitHub
→ Settings
→ SSH and GPG Keys
→ New SSH Key


Paste:

ssh-ed25519 AAAA...


Save.

9. Test GitHub Authentication

Run:

ssh -T git@github.com

Explanation
Part	Meaningssh	Open SSH connection
-T	No interactive shell
git@github.com	GitHub SSH endpoint

Expected:

Hi username! You've successfully authenticated...


That means GitHub recognises your key.

10. Clone Repositories Using SSH

Instead of:

https://github.com/user/project.git


Use:

git@github.com:user/project.git


Example:

git clone git@github.com:myuser/myrepo.git

Explanation
Part	Meaninggit clone	Download repo
git@github.com	SSH connection
myuser/myrepo.git	Repository

No passwords required.

11. Add Key to Linux Server

If available:

ssh-copy-id user@server-ip


Example:

ssh-copy-id root@192.168.1.50

Explanation
Part	Meaningssh-copy-id	Installs your public key on server
root	Server username
192.168.1.50	Server IP

This copies:

id_ed25519.pub


into:

~/.ssh/authorized_keys


on the server.

12. Connect to Server

Run:

ssh root@192.168.1.50

Explanation
Part	Meaningssh	Connect via SSH
root	Username
192.168.1.50	Server address

If the key is installed correctly:

✅ No server password needed.

13. Make Servers Easier to Remember

Edit:

nano ~/.ssh/config


Add:

Host production
    HostName 192.168.1.50
    User root
    IdentityFile ~/.ssh/id_ed25519

Explanation
Setting	MeaningHost production	Nickname
HostName	Real IP/domain
User	Username
IdentityFile	SSH key

Now instead of:

ssh root@192.168.1.50


you simply do:

ssh production


This is what most developers use.

Commands You'll Actually Use Daily
Connect to a server
ssh production


or

ssh root@server-ip

Clone repository
git clone git@github.com:user/repo.git

Push code
git push

Pull updates
git pull

The 3 Files You Should Remember
~/.ssh/id_ed25519


Your private key (never share)

~/.ssh/id_ed25519.pub


Your public key (share with GitHub/servers)

~/.ssh/config


Your SSH settings

Quick Mental Model

One Mac ↓

One SSH key pair (id_ed25519 + id_ed25519.pub)

↓

Public key goes to:

GitHub ✅
VPS ✅
AWS machine ✅
DigitalOcean ✅
Ubuntu server ✅
Raspberry Pi ✅

You normally keep the same key for years and simply add the public key wherever you need access. This is exactly how most software engineers manage GitHub and server access.
