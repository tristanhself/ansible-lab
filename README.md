# Bootstrapping - Control Node (Management Workstation)






source ~/virtualenvs/ansible-venv/bin/activate

# Bootstrapping - Managed Nodes

How to bootstrap the hosts for management with Ansible.

## Create Ansible User

### Fedora/RedHat

ansible -i hosts -l web01 -u web01-admin -k -b -K -m command -a "useradd ansible"
ansible -i hosts -l web01 -u web01-admin -k -b -K -m shell -a "echo password | passwd --stdin ansible"

ansible -i ~/projects/ansible-lab/hosts/99-hosts web01 -u web01-admin -k -b -K -m command -a "useradd ansible"
ansible -i ~/projects/ansible-lab/hosts/99-hosts web01 -u web01-admin -k -b -K -m shell -a "echo <password> | passwd --stdin ansible"

(Not ideal because you are potentially exposing the password to the history of the "Control Node".)

### Ubuntu

ansible -i hosts -l web02 -u web02-admin  -k -b -K -m command -a "useradd -m ansible"
ansible -i hosts -l web02 -u web02-admin -k -b -K -m shell -a "echo :ansiblepassword | chpasswd"

ansible -i ~/projects/ansible-lab/hosts/99-hosts web02 -u web02-admin -k -b -K -m command -a "useradd -m ansible"
ansible -i ~/projects/ansible-lab/hosts/99-hosts web02 -u web02-admin -k -b -K -m shell -a "echo ansible:<password> | chpasswd"
ansible -i ~/projects/ansible-lab/hosts/99-hosts web02 -u web02-admin -k -b -K -m command -a "usermod -s /bin/bash ansible"

(Not ideal because you are potentially exposing the password to the history of the "Control Node".)

However it is also recommended to use the "User" module instead.

ansible -i ~/projects/ansible-lab/hosts/99-hosts web02 -u web02-admin -k -b -K -m user -a "name=ansible shell=/bin/bash state=present"

### Verify 

Check the accounts are created ok and work.

ansible -i ~/projects/ansible-lab/hosts/99-hosts all -u ansible -k -m command -a "whoami"

Check we can't yet access root:

ansible -i ~/projects/ansible-lab/hosts/99-hosts all -u ansible -k -m command -a "ls -l /root"

### Setup Sudo for Ansible

To ensure Ansible account can gain privilege escalation to perform certain tasks, we add the following, of course we need to do this as our admin account rather than the ansible account
at this stage.

ansible -i ~/projects/ansible-lab/hosts/99-hosts web01 -u web01-admin -k -b -K -m copy -a 'content="ansible ALL=(ALL) NOPASSWD: ALL" dest=/etc/sudoers.d/ansible'
ansible -i ~/projects/ansible-lab/hosts/99-hosts web02 -u web02-admin -k -b -K -m copy -a 'content="ansible ALL=(ALL) NOPASSWD: ALL" dest=/etc/sudoers.d/ansible'

Now if we try again:

ansible -i ~/projects/ansible-lab/hosts/99-hosts all -u ansible -k -m command -a "whoami"

ansible -i ~/projects/ansible-lab/hosts/99-hosts all -u ansible -k -b -m command -a "ls -l /root"

We should be able to list the root home directory, notice the "-b" meaning we want to "become", but also notice we are omitting the "-K" which means we are not asking for the "Become" password, because in this case we don't need it, being we added the exception in the sudoers.d file to allow the "Ansible" account to sudo without needing a password.

# Generate and Distribute SSH Keypair

Will create you a SSH Keypair in as Private Key: ~/.ssh/id_ed25519, Public Key: ~/.ssh/id_ed25519.pub:

ssh-keygen

We now distribute this to the Managed Nodes with the following, this will copy our SSH public key into the ansible user's key store on the Managed Nodes. So we can then manage the nodes without having to keep re-entering the SSH password each time.

ssh-copy-id -i ~/.ssh/id_ed25519.pub ansible@192.168.102.201
ssh-copy-id -i ~/.ssh/id_ed25519.pub ansible@192.168.102.202

Now we check we can login without an SSH password:

ssh -i ~/.ssh/id_ed25519 ansible@192.168.102.201
ssh -i ~/.ssh/id_ed25519 ansible@192.168.102.202

We of course don't want to have to specify the SSH key file each time, so we can load this into our SSH Agent.

eval $(ssh-agent -s)
ssh-add ~/.ssh/id_ed25519
ssh-add -l

You should now see the identity there, which means when using Ansible, assuming we are not specifically saying to use an SSH Password it will automatically pick the SSH Key and use that for the SSH key based connection. Let's try that now.

ansible -i ~/projects/ansible-lab/hosts/99-hosts all -u ansible -m command -a "whoami"

Assuming this runs, and shows the user "ansible" and doesn't ask for a password, it means the SSH Key based authentication is working successfully. Let's now check the ability to sudo (privilege escalate) too.

ansible -i ~/projects/ansible-lab/hosts/99-hosts all -u ansible -b -m command -a "ls -l /root"

And again, if we get the root user's home directory listed, its worked as expected.