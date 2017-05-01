Creates two vagrant boxes, one with CentOS 6, and the other with CentOS 7, to test dm-crypt on kernels 2.6.x and 3.2.x.

# Setup
Each box has two virtual hard disks attached to them. The first will be encypted with `dm-crypt` using `cryptsetup`. The second is unencrypted, and will be used to compare file read and write performance.

## Vagrant
The Vagrantfile specifies two VMs

- the first running CentOS 6 with 2 disks attached to it. Note that the box specified is `rhel` which is the name I have give to my CentOS 6 base box.

```ruby
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
```

- the second running CentOS 7 with 2 disks attached to it. `centos7` is the name I have give to my CentOS 7 base box.

```ruby
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
```

Start boxes using the following command
```
$ vagrant up
```
Once the servers are up, SSH into each of them and run the `lsblk` command to list available block devices.

- CentOS 6

```bash
[vagrant@vagrant-centos64 ~]$ lsblk
NAME   MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sda      8:0    0    8G  0 disk
└─sda1   8:1    0    8G  0 part /
sdc      8:32   0    1G  0 disk
sdb      8:16   0  1.2G  0 disk
└─sdb1   8:17   0  1.2G  0 part [SWAP]
```

- CentOS 7

```bash
[vagrant@localhost ~]$ lsblk
NAME            MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sda               8:0    0    8G  0 disk
├─sda1            8:1    0  500M  0 part /boot
└─sda2            8:2    0  7.5G  0 part
  ├─centos-swap 253:0    0  820M  0 lvm  [SWAP]
  └─centos-root 253:1    0  6.7G  0 lvm  /
sdb               8:16   0    1G  0 disk
sdc               8:32   0    1G  0 disk
sr0              11:0    1 1024M  0 rom
```

## dm-crypt and LUKS
`cryptsetup` is a command line interface for `dm-crypt`, with Linux Unified Key Setup (LUKS), to manage encrypted devices.

The following commands will work on both centos6 and centos7 boxes.

Wipe the block device before formatting it for use

```bash
[vagrant@vagrant-centos64 ~]$ sudo cat /dev/zero > /dev/sdb
```
Create the LUKS container
```bash
[vagrant@vagrant-centos64 ~]$ sudo cryptsetup luksFormat /dev/sdb
```
This will overwrite any existing data. `cryptsetup` will ask for confirmation before proceeding. It will also prompt to set a LUKS passphrase for the device.

`
NOTE: LUKS allows up to 8 different keys per LUKS partition. It is a good practice to have a backup key in case the first key is corrupted or compromised.
`

Map the container to a device. It is recommended to create a mapper in `/dev/mapper`
```bash
[vagrant@vagrant-centos64 ~]$ sudo cryptsetup luksOpen /dev/sdb encrypted
```
`cryptsetup` will prompt for the passphrase for the container. It will create a mapper name `encrypted` in `/dev/mapper`.

Create a filesystem on the mapped container, and mount it.
```bash
[vagrant@vagrant-centos64 ~]$ sudo mkfs.xfs /dev/mapper/encrypted
[vagrant@vagrant-centos64 ~]$ sudo mkdir -p /mnt/encrypted
[vagrant@vagrant-centos64 ~]$ sudo mount /dev/mapper/encrypted /mnt/encrypted
```
Verify the mounted filesystem
```bash
[vagrant@vagrant-centos64 ~]$ lsblk
NAME               MAJ:MIN RM  SIZE RO TYPE  MOUNTPOINT
sdb                  8:16   0    1G  0 disk  
└─encrypted (dm-0) 253:0    0 1022M  0 crypt /mnt/encrypted
sdc                  8:32   0    1G  0 disk  
sda                  8:0    0    8G  0 disk  
└─sda1               8:1    0    8G  0 part  /
```
Use the second attached device to create an unencrypted file system for comparison testing.
```bash
[vagrant@vagrant-centos64 ~]$ sudo mkfs.xfs /dev/sdc
[vagrant@vagrant-centos64 ~]$ sudo mkdir -p /mnt/unencrypted
[vagrant@vagrant-centos64 ~]$ sudo mount /dev/sdc /mnt/unencrypted
[vagrant@vagrant-centos64 ~]$ lsblk
NAME               MAJ:MIN RM  SIZE RO TYPE  MOUNTPOINT
sdb                  8:16   0    1G  0 disk  
└─encrypted (dm-0) 253:0    0 1022M  0 crypt /mnt/encrypted
sdc                  8:32   0    1G  0 disk  /mnt/unencrypted
sda                  8:0    0    8G  0 disk  
└─sda1               8:1    0    8G  0 part  /
```

## File I/O Metrics
The `dd` command can be used for simple benchmarking by copying a file to encrypted and unencrypted partitions and timing the operations. Different byte size and count options can be specified to vary the tests.

```bash
[root@vagrant-centos64 ~]# time sh -c "dd bs=1M count=256 if=/dev/zero of=/mnt/encrypted/test conv=fdatasync"; rm -f /mnt/encrypted/test
256+0 records in
256+0 records out
268435456 bytes (268 MB) copied, 2.52194 s, 106 MB/s

real	0m2.530s
user	0m0.000s
sys	  0m0.158s
```

By default, the `dd` command does not ask the operating system to complete write (sync) the data before exiting. For some applications, it is necessary to completely flush the data to disk before confirming the write for data integrity requirements. An example is Oracle's "log file sync" wait event that is triggered when a user session issues a commit. The commit is not marked complete until the log writer (LGWR) flushes content in the log buffer to disk (to insure recovery).

The `fdatasync` option ensures that the `dd` command does not exit until it writes the data to disk. More information about various options for the `dd` command is available in the [dd man page](http://linux.die.net/man/1/dd).

For comparison, run the same command to create the file on the unencrypted volume.
```bash
[root@vagrant-centos64 ~]# time sh -c "dd bs=1M count=256 if=/dev/zero of=/mnt/unencrypted/test conv=fdatasync"; rm -f /mnt/unencrypted/test
256+0 records in
256+0 records out
268435456 bytes (268 MB) copied, 0.30591 s, 877 MB/s

real	0m0.314s
user	0m0.003s
sys	  0m0.188s
```

The commands above write 256 input blocks of 1MB size each to an output file on the partitions. From the results, it can be inferred that writes to the encrypted volume perform at about 1/8<sup>th</sup> the speed compared to writes on the unencrypted volume. Command parameters can be tweaked to collect statistics for different write operations.

Read performance can also be measured with the same command. In the previous commands, remove the `rm` command at the end to keep the files on the volumes. Then copy the file to `/dev/null` and time the operation

```bash
[root@vagrant-centos64 ~]# time sh -c "dd bs=1M count=256 if=/mnt/encrypted/test of=/dev/null"
256+0 records in
256+0 records out
268435456 bytes (268 MB) copied, 0.0438085 s, 6.1 GB/s

real	0m0.047s
user	0m0.002s
sys	  0m0.046s
```

Run the same commands on the CentOS 7 system to compare results again the different kernel version on 6 & 7.

# References
- [Secret Messages](http://nnc3.com/LM10/Magazine/Archive/2005/61/065-071_encrypt/article.html)
- [cryptsetup](https://gitlab.com/cryptsetup/cryptsetup)
- [dd man page](http://linux.die.net/man/1/dd)
