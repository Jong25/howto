# 1. Install docker

1. install docker-engine

[Install Docker Engine on Ubuntu](https://docs.docker.com/engine/install/ubuntu/)

```bash
sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

2. add user to privilege group

[Linux post-installation steps for Docker Engine](https://docs.docker.com/engine/install/linux-postinstall/)

```bash
sudo groupadd docker
sudo usermod -aG docker $USER
```

3. reboot and check if docker daemon is running

```bash
sudo systemctl list-sockets | grep docker.service
# if not running
sudo systemctl start docker.service
```

# 2. Make rootfs image

1. download reference rootfs from AWS

[Ubuntu rootfs customization — Firefly Wiki](https://wiki.t-firefly.com/en/Firefly-Linux-Guide/custom_ubuntu_rootfs.html)

```bash
curl -fsSL -o ref-rootfs.ext4 https://s3.amazonaws.com/spec.ccfc.min/img/quickstart_guide/x86_64/rootfs/bionic.rootfs.ext4
```

2. make dummy ext4 file (size 16G)

```bash
dd if=/dev/zero of=linuxroot.img bs=1M count=16K
sudo mkfs.ext4 linuxroot.img
```

3. copy files

```bash
sudo mkdir /mnt/ref-rootfs && sudo mount ref-rootfs.ext4 /mnt/ref-rootfs
sudo mkdir /mnt/linuxroot && sudo mount linuxroot.img /mnt/linuxroot
sudo cp -rfp /mnt/ref-rootfs/*  /mnt/linuxroot/
sudo umount /mnt/ref-rootfs
```

4. *additional
    a. download required files, packages, …
    
    ```bash
    sudo mount linuxroot.img /mnt/linuxroot
    vim /mnt/linuxroot/etc/resolv.conf
    # search 8.8.8.8, nameserver 8.8.8.8
    chroot /mnt/linuxroot
    # do what you want(apt install, ...)
    ```
    
    b. shrink image size to fit
    
    ```bash
    e2fsck -p -f linuxroot.img
    resize2fs  -M linuxroot.img
    ```
    

# 3. Install firecracker

1. build from source

[GitHub - firecracker-microvm/firecracker: Secure and fast microVMs for serverless computing.](https://github.com/firecracker-microvm/firecracker)

```bash
git clone https://github.com/firecracker-microvm/firecracker
cd firecracker
tools/devtool build
toolchain="$(uname -m)-unknown-linux-musl"
```

2. configure setting

[firecracker/getting-started.md at main · firecracker-microvm/firecracker](https://github.com/firecracker-microvm/firecracker/blob/main/docs/getting-started.md)

```bash
vim config.json
```

```json
{
  "boot-source": {
    "kernel_image_path": "$path_to_vmlinux$",
    "boot_args": "console=ttyS0 earlyprintk=ttyS0 ftrace_dump_on_oops nokaslr",
    "initrd_path": "$path_to_initrd$"
  },
  "drives": [
    {
      "drive_id": "rootfs",
      "path_on_host": "$path_to_rootfs_image$",
      "is_root_device": true,
      "partuuid": null,
      "is_read_only": false,
      "cache_type": "Unsafe",
      "io_engine": "Sync",
      "rate_limiter": null
    }
  ],
  "machine-config": {
    "vcpu_count": 8,
    "mem_size_mib": 8192,
    "smt": false,
    "track_dirty_pages": false
  },
  "balloon": null,
  "network-interfaces": [],
  "vsock": null,
  "logger": null,
  "metrics": null,
  "mmds-config": null
}
```

3. run firecracker

```bash
rm -f /tmp/firecracker.socket
# firecracker is in build/cargo_target/x86_64-unknown-linux-musl/debug/firecracker
./firecracker --api-sock /tmp/firecracker.socket --config-file <path_to_the_configuration_file>
```
