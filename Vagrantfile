Vagrant.configure("2") do |config|
  config.vagrant.plugins = "vagrant-libvirt"
  config.vm.provider "libvirt" do |libvirt|
    libvirt.memory = "4096"
  end
  config.vm.box = "generic/rocky8"
  config.vm.define "nfs-server" do |server|
    server.vm.hostname = "nfs-server"
    server.vm.network :private_network, :ip => "10.0.0.10"
  end
  config.vm.define "nfs-client" do |client|
    client.vm.hostname = "nfs-client"
    client.vm.network :private_network, :ip => "10.0.0.11"
  end
  config.vm.provision "ansible" do |ansible|
    ansible.verbose = "v"
    ansible.playbook = "playbook.yml"
#    ansible.ask_vault_pass = true
  end
end
