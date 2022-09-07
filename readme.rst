nfs-lab
*******
This repository represents a virtual network to practice setting
up a file sharing using nfs.

It consists of two nodes: a nfs server and a nfs client.

To use this, you'll need Vagrant installed. Run ``vagrant up`` to
create the VMs and hook them up into a network. Run ``vagrant ssh
nfs-server`` to connect to the server and issue commands. Run
``vagrant ssh nfs-client`` to connect to the client and issue
commands.

I have no idea what I'm doing, but I'm rapidly learning Vagrant
and Ansible, so any tips are appreciated.
