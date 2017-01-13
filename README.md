# todo:

- ruby installer / general rails setup
- nginx configs
- research way around --ask-vault-pass
- look into the `authorized_key` task's `key=` bit. see note below

stuff to do when starting with this repo:

**if you're following the walkthrough below, skip this section!**

- use the included example `Vagrantfile` if you want Ubuntu 16.06 and for the IPs in the various files to line up and stuff.
- check out `vagrant ssh-config` to get the path to the `IdentityFile` vagrant is using, if you're not using your own key.
- if moving your boxes to a USB, follow steps [like this](https://emptysqua.re/blog/moving-virtualbox-and-vagrant-to-an-external-drive/)
    - change VirtualBox preferences to reflect the move as well preferences > general > default machine folder
- on a new machine, run `vagrant status` to make sure your box is there. then `vagrant up` to have it set up.

--------------

# Walkthrough

## Vagrant setup

- install vagrant locally (look this up for your system).
- install VirtualBox, VMware or a similar 'provider' (I use VirtualBox)
- run `$ vagrant init [BOX NAME]` inside the directory where you want to do all your hosting/testing. maybe `~/` ? where `[BOX NAME]` should be replaced by the os/box desired, like `vagrant init ubuntu/trusty64` or `vagrant init hashicorp/precise64`. Check here for more options: https://atlas.hashicorp.com/boxes/search. This will create a `Vagrantfile` in this directory.
- run `$ vagrant up` in the same directory as above. The box will be downloaded if not already present on your system. This might take a bit. Go pour yourself a glass of peanut butter or whatever.
- check out the generated `Vagrantfile`. Uncomment the `config.vm.network "private_network", ip: "192.168.33.10"` line and change the IP to something you want to use (or leave it alone, but note it down for later).  If you change this file, you'll need to issue `$ vagrant reload` to restart the box with the new configs.
- run `$ vagrant ssh-config` to see the settings that will be used by default to ssh into this box if you run `vagrant ssh`. Note there is an IdentityFile path here. This private key will get you ssh access to the box when _not_ using the `vagrant ssh` command. You'll need that later.

## Ansible setup (specifically for use with this repo)

- install ansible locally (look this up for your system).
- the explanation for all ansible commands is a bit outside the scope of this project, so definitely check out the ansible basics to get acquainted with what's going on with the different commands and arguments.
- first off, make sure the IP and path to private key file is correct (from the vagrant details above) in `hosts/vagrant-box`. Use the IP you have in your `Vagrantfile` and the private key path from the `vagrant ssh-config` command.
- once that's all lined up, when you run `$ ansible-playbook -i hosts test_connection.yml` you should get an `OK` response or two. This means the ansible host data is aligned with your vagrant box. **That's about the end of the setup needed**. If you've got SSH, you've got everything you need for ansible.

- if you don't already have an rsa keypair, generate one now (or generate a new one specifically to use with this. for now, for testing this with vagrant, I've created a key and stored it in dropbox which is ok because I'm not doing anything secret with it).
- also, create a `vars` directory at the top level of this repo so the next command doesn't fail ;)
- run `$ ansible-vault create vars/passwords.yml` (this file is not checked in to git). the `vars` folder has different variables that will be used in the below playbooks.
- within, define `admin_user_password`, `deploy_user_password` and `common_public_key` like this:

```
admin_user_password: THE_PASSWORD_FOR_SERVERS_ADMIN_USER
deploy_user_password: THE_PASSWORD_FOR_SERVERS_DEPLOY_USER
common_public_key: "PASTE_YOUR_ID_RSA_PUB_HERE_IN_THESE_QUOTES"
```

_note: This `common_public_key` bit above is likely far from ideal or even bad... Now that I look back at the `authorized_key` [task documentation](http://docs.ansible.com/ansible/authorized_key_module.html) that this variable is eventually used for in the below playbooks, it looks like there is a better way to gather that information, like so:_ `"{{ lookup('file', '/home/some_user/.ssh/id_rsa.pub') }}"`. _I should probably look into that._

Ansible setup is now complete. You can run the playbooks included here. The explanations for these playbooks is below.

## Explanation of the various playbooks included here

- server-setup/ _folder_
    - package-setup: insures apt is updated, then installs some basic/common packages (see the `with_items` list to see which ones).
    - create-admin-user: creates a sudoy `admin` user in the `sudo` group and adds a public key for access.
    - create-deploy-user: creates a sudoy `deploy` user in the `www-data` group and adds same key as above.
    - modify-var-www-for-deploy: sort of does what it says. modifies the `www-data` folder so the `deploy` user can use it.
    - nginx/ _folder_
        - nginx-setup: installs nginx and copies over the default config file from the template in this repo.
        - handler: restarts nginx, when needed.
- combo playbooks, _at the top-level_
    - server-setup-main: runs the package-setup playbook.
    - prepare-for-deploy-user: runs create-deploy-user and modify-var-www-for-deploy.
    - nginx-webserver-setup: runs package-setup and nginx-setup.

Given the playbooks/tasks currently in existence, if you do this in this order:

- `ansible-playbook -i hosts server-setup/create-admin-user --ask-vault-pass`
- `ansible-playbook -i hosts prepare-for-deploy-user.yml --ask-vault-pass`
- `ansible-playbook -i hosts nginx-webserver-setup-main.yml`

You'll end up with a server that has `admin` and `deploy` users, some sane apt packages, nginx installed and a `www-data` directory ready to go. ...Then you'll have to do all the other hard stuff like install ruby/rails and all your dependencies and all the nginx configs XD.  
At least until they're added to this repo.


## Gotchas

- **For any commands that read from the vault-encrypted passwords.yml file, you'll need to pass `--ask-vault-pass` or else the command will fail with 'Decryption failed' and confuse you**. There's probably a way around this but I haven't looked into it yet.
