# -*- mode: ruby -*-
# vi: set ft=ruby :

# Vagrantfile API/syntax version. Don't touch unless you know what you're doing!
VAGRANTFILE_API_VERSION = "2"

Vagrant.configure(VAGRANTFILE_API_VERSION) do |config|
  # Disks to encrypt
  c6_encrypted_disk = './tmp/c6_encrypted_disk.vdi'
  c6_unencrypted_disk = './tmp/c6_unencrypted_disk.vdi'
  c7_encrypted_disk = './tmp/c7_encrypted_disk.vdi'
  c7_unencrypted_disk = './tmp/c7_unencrypted_disk.vdi'

  # CentOS 6
  config.vm.define "centos6" do |centos6|
    centos6.vm.box = "rhel"
    centos6.vm.provider "virtualbox" do |vb|
      # Add disk to encrypt
      unless File.exist?(c6_encrypted_disk)
        vb.customize ['createhd', '--filename', c6_encrypted_disk, '--size', 1024]
        # Attach encrypted disk
        vb.customize ['storageattach', :id, '--storagectl', 'SATA', '--port', 1, '--device', 0, '--type', 'hdd', '--medium', c6_encrypted_disk]
      end

      # Add unencrypted disk
      unless File.exist?(c6_unencrypted_disk)
        vb.customize ['createhd', '--filename', c6_unencrypted_disk, '--size', 1024]
        # Attach unencrypted disk
        vb.customize ['storageattach', :id, '--storagectl', 'SATA', '--port', 2, '--device', 0, '--type', 'hdd', '--medium', c6_unencrypted_disk]
      end
    end
  end

  # CentOS 7
  config.vm.define "centos7" do |centos7|
    centos7.vm.box = "centos7"
    centos7.vm.provider "virtualbox" do |vb|
      # Add disk to encrypt
      unless File.exist?(c7_encrypted_disk)
        vb.customize ['createhd', '--filename', c7_encrypted_disk, '--size', 1024]
        # Attach encrypted disk
        vb.customize ['storageattach', :id, '--storagectl', 'SATA', '--port', 1, '--device', 0, '--type', 'hdd', '--medium', c7_encrypted_disk]
      end

      # Add unencrypted disk
      unless File.exist?(c7_unencrypted_disk)
        vb.customize ['createhd', '--filename', c7_unencrypted_disk, '--size', 1024]
        # Attach unencrypted disk
        vb.customize ['storageattach', :id, '--storagectl', 'SATA', '--port', 2, '--device', 0, '--type', 'hdd', '--medium', c7_unencrypted_disk]
      end
    end
  end
end
