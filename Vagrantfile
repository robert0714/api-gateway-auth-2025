# -*- mode: ruby -*-
# vi: set ft=ruby :

# Check for missing plugins
required_plugins = %w(vagrant-hostmanager)
plugin_installed = false
required_plugins.each do |plugin|
  unless Vagrant.has_plugin?(plugin)
    system "vagrant plugin install #{plugin}"
    plugin_installed = true
  end
end

# If new plugins installed, restart Vagrant process
if plugin_installed === true
  exec "vagrant #{ARGV.join' '}"
end

Vagrant.configure("2") do |config|
  if (/cygwin|mswin|mingw|bccwin|wince|emx/ =~ RUBY_PLATFORM) != nil
    config.vm.synced_folder ".", "/vagrant", mount_options: ["dmode=700,fmode=600"]
  else
    config.vm.synced_folder ".", "/vagrant"
  end
  # vagrant-hostmanager options
  config.hostmanager.enabled = true
  config.hostmanager.manage_host = false # Your permission would be denied
  config.hostmanager.manage_guest = true

  # Forward ssh agent to easily ssh into the different machines
  config.ssh.forward_agent = true

  config.vm.box = "bento/ubuntu-22.04"
  config.vm.hostname = "sam.local.dev"

  config.vm.network :private_network, ip: "10.100.198.101"   
  config.vm.provider "virtualbox" do |vb|
    vb.gui = false
    vb.name = "sam"
    vb.memory = "5120"
    vb.cpus = "2"
  end
  config.vm.provision :docker 
  config.vm.provision :hostmanager
  config.hostmanager.aliases = %w(beans.rock dukes.rock)
  if Vagrant.has_plugin?("vagrant-cachier")
    config.cache.scope = :box
  end
  if Vagrant.has_plugin?("vagrant-vbguest")
    config.vbguest.auto_update = false
    config.vbguest.no_install = false
    config.vbguest.no_remote = false
  end
end