# Nix Installer for Synology NAS
This is a fork of the [Determinate Systems nix-installer](https://github.com/DeterminateSystems/nix-installer) for installing nix on Synology NAS devices.

## Disclaimer
This tool has been tested on a Synology DS920 running `DSM 7.2.1-69057 Update 5`.
```console
$ uname -a
Linux DS920plus 4.4.302+ #69057 SMP Fri Jan 12 17:02:28 CST 2024 x86_64 GNU/Linux synology_geminilake_920+
```

Use at your own risk. Do not use this on any other platforms, use the upstream DetSys installer instead.

## Steps 


### Setup /nix

Root volume doesn't have enough space. Need to put nix store on data volume and bind mount to /nix.

See https://stackoverflow.com/a/34966233
> On Linux, bind mounts can be used instead of symlink for this purpose (e.g., `mount -o bind /data/nix/store /nix/store').


On the NAS
```bash
# Find data volume
df -h
# Volume is /volume1

# Create directory in volume
sudo mkdir /nix
sudo chmod 755 /nix
sudo mkdir /volume1/nix
sudo chmod 755 /volume1/nix

# Bind mount
mount -o bind /volume1/nix /nix
# Note that /etc/fstab will get reset on boot, so don't bother creating an entry. More on this later 
```

### Build installer
On another other device
```bash
# Clone this repo
git clone https://github.com/adam-gaia/synology-nix-installer.git

# Build static binary
nix build .#packages.x86_64-linux.nix-installer-static -L

# Copy bin to nas
rsync ./result/bin/nix-installer NAS_USERNAME@NAS_IP:~/ # I had rsync already enabled on my nas and not scp
```

### Install nix
On NAS again
```bash
# Need to disable syscall filtering
# See https://github.com/DeterminateSystems/nix-installer/issues/324#issuecomment-1479536235
NIX_INSTALLER_EXTRA_CONF='filter-syscalls = false' ./nix-installer install
# Will prompt for sudo password
```

### Enable daemon
One of the patches made in this fork stops the nix daemon from starting automatically. (The DetSys installer supportes systemd 220, but DSM7.2.7 ships with systemd 219, which doesnt support the `--now` flag)
```
sudo systemctl start nix-daemon	
# More on autostarting this later
```

### Test
```
source /nix/var/nix/profiles/default/etc/profile.d/nix-daemon.sh # TODO: add to bashrc!

nix run nixpkgs#hello
```

### Startup scripts
System configuration in /etc gets reset on reboot, which makes tinkering annoying.
 The only way I could figure out how to auto-run scripts on startup was in the UI.
We'll make two user-defined scripts

- Control Panel -> Task Scheduler -> Create -> Triggered Task -> User-defined script


#### Make the /nix mount persistant
- Task: Bind mount nix
- User: root
- Event: Boot-up
- Pre-task: None

Script:
```
mount -o bind /volume1/nix /nix	
```

#### Auto start nix-daemon service
- Task: Start nix daemon
- User: root
- Event: Boot-up
- Pre-task: Bind mount nix

Script:
```
systemctl start nix-daemon	
```

# TODO: it would be cool to add a nix garbage collect user-defined script

