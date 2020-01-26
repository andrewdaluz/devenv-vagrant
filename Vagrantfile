# -*- mode: ruby -*-
# vi: set ft=ruby :

# Determine persistent and source paths
if File.symlink?(__FILE__)
  # Assume we are already in the persistent path
  persistent_path = "./"
  source_path = "./source/"
  persistent_disk_path = ""
else
  # Assume we are in the source path
  persistent_path = "./persistent/"
  source_path = "./"
  persistent_disk_path = "persistent/"
  print "\e[31m"+"Warning: "+"\e[33m"+"This is designed to work better when calling vagrant from the persistent directory where Vagrantfile has been symlinked to the source directory."+"\e[0m"
end

# include local variables, if present
require_relative "Vagrantfile.default"
require "#{source_path}Vagrantfile.config.rb" if File.file?("#{source_path}Vagrantfile.config.rb")
require "#{source_path}Vagrantfile.local.rb" if File.file?("#{source_path}Vagrantfile.local.rb")
require "#{persistent_path}Vagrantfile.config.rb" if File.file?("#{persistent_path}Vagrantfile.config.rb")
require "#{persistent_path}Vagrantfile.local.rb" if File.file?("#{persistent_path}Vagrantfile.local.rb")

# include plugin files
require_relative 'plugin/PluginUtility'
require_relative 'plugin/PluginPersistDisk'
require_relative 'plugin/PluginMutagen'
require_relative 'plugin/PluginHostmanagerHelper'

# indicate which versions this has been tested with
Vagrant.require_version ">= 2.2.5", "< 2.3.0"

Vagrant.configure('2') do |config|
  
  config.vagrant.plugins = ["vagrant-hostmanager"]
  
  # vagrant-hostmanager configuration
  config.hostmanager.enabled = true
  config.hostmanager.manage_host = true
  config.hostmanager.manage_guest = false
  config.hostmanager.aliases = $dev_additional_hostnames
  
  config.vm.define "#{$dev_machine_name}" do |config|
    
    config.vm.box = "bento/centos-7"
    config.vm.hostname = "#{$dev_machine_name}"
    config.vm.network :private_network, type: "dhcp"
    config.hostmanager.ignore_private_ip = false
    
    # Resolve guest DHCP assigned IP
    cached_addresses = {}
    config.hostmanager.ip_resolver = proc do |vm, resolving_vm|
      PluginHostmanagerHelper.vbIpResolver(vm, cached_addresses)
    end
    PluginHostmanagerHelper.vmUpTrigger(config, $dev_machine_name)
    
    # After provisioning change vagrant ssh to connect via
    # the hostname on the host_only network and use the 
    # www-data user when logging into the box
    if PluginUtility.vmIsProvisioned("#{$dev_machine_name}")
      config.ssh.private_key_path = "#{$ssh_private_key}"
      config.ssh.insert_key = false
      config.ssh.host = "#{$dev_machine_name}"
      config.ssh.port = 22
      config.ssh.username = 'www-data'
    end
    config.ssh.forward_agent = true
    config.vm.graceful_halt_timeout = 120
    # Disable the default synced folder
    config.vm.synced_folder '.', '/vagrant', disabled: true
    
    # Persistent Disk variable settings
    persistentDisks = Array.new
    nextPersistContPort = 1
    $persistent_disks.each {|disk|
      persistentDisks.push(
        {
          "description" => disk["description"],
          "diskControllerName" => "SATA Controller",
          "workingDirectory" => Dir.pwd,
          "persistDiskPath" => "#{persistent_disk_path}#{disk["persistDiskPath"]}",
          "persistDiskSizeGb" => disk["persistDiskSizeGb"],
          "persistContPort" => nextPersistContPort,
          "persistContDev" => 0
        }
      )
      nextPersistContPort += 1
    }
    
    # configure RAM, CPUs and other machine settings
    config.vm.provider "virtualbox" do |vb|
      vb.name = "#{$dev_machine_name}"
      vb.memory = $dev_vm_ram
      vb.cpus = $dev_vm_cpus
      # Adjust vram size to expected min
      vb.customize ["modifyvm", :id, "--vram", "16"]
      # Prevent VirtualBox from interfering with host audio stack
      vb.customize ["modifyvm", :id, "--audio", "none"]
      # Disable VirtualBox Remote Display
      vb.customize ["modifyvm", :id, "--vrde", "off"]
      
      persistentDisks.each {|disk| 
        PluginPersistDisk.vmCreate(
          vb, #vb
          disk["diskControllerName"], #diskControllerName
          persistentDisks.count+1, #diskControllerPortCount
          disk["persistContPort"], #persistContPort
          disk["persistContDev"], #persistContDev
          disk["workingDirectory"], #workingDirectory
          disk["persistDiskPath"], #persistDiskPath
          disk["persistDiskSizeGb"] #persistDiskSizeGb
        )
      }
      
    end
    
    # Provision
    config.vm.provision "ansible" do |ansible|
      ansible.playbook = "#{source_path}provisioning/build.yml"
      ansible.extra_vars = { host_zoneinfo: File.readlink('/etc/localtime') }
      ansible.compatibility_mode = "2.0"
      ansible.force_remote_user = false
      ansible.extra_vars = {
        host_zoneinfo: File.readlink('/etc/localtime'),
        mysql_root_pw: $dev_mysql_root_pw,
        sSH_PUBLIC_KEY_PATHS: $ssh_public_key_paths
      }
    end
    
    # Triggers
    persistentDisks.each {|disk| 
      PluginPersistDisk.vmUp(config, $dev_machine_name, disk["diskControllerName"], disk["persistContPort"], disk["persistContDev"], disk["persistDiskPath"])
    }
    #PluginMutagen.vmUp(config)
    
    #PluginMutagen.vmHalt(config)
    persistentDisks.each {|disk| 
      PluginPersistDisk.vmHalt(config, $dev_machine_name, disk["diskControllerName"], disk["persistContPort"], disk["persistContDev"])
    }
    
    #PluginMutagen.vmDestroy(config)
    persistentDisks.each {|disk| 
      PluginPersistDisk.vmDestroy(config, $dev_machine_name, disk["diskControllerName"], disk["persistContPort"], disk["persistContDev"])
    }
    
  end
  
end

