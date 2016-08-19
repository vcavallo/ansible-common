# todo:

- ruby installer / general rails setup

This setup is designed for Ubuntu 16.06 only for now.

- you will need to `ansible-vault create vars/passwords.yml` (this file is not checked in to git)
- within, define `admin_user_password`, `deploy_user_password` and `common_public_key` (first create an rsa keypair. For now, for testing this with vagrant, I've created a key and stored it in dropbox which is ok because I'm not doing anything secret with it)

other stuff to do:

- use the included example `Vagrantfile` if you want the IPs to line up and stuff
- check out `vagrant ssh-config` to get the path to the `IdentityFile` vagrant is using, if you're not using your own key.
- if moving your boxes to a USB, follow steps [like this](https://emptysqua.re/blog/moving-virtualbox-and-vagrant-to-an-external-drive/)
    - change VirtualBox preferences to reflect the move as well preferences > general > default machine folder

on a new machine, run `vagrant status` to make sure your box is there. then `vagrant up` to have it set up.
