# DEV & LINUX RUNBOOK

CLI-first operational handbook for system administration, containerization, toolchain setups, and kernel-level troubleshooting.

```
+-------------------------------------------------------------------------+
| QUICK CHEATSHEET                                                        |
|-------------------------------------------------------------------------|
| Drop RAM Cache         | lp                                             |
| Docker Purge           | docker system prune -a --volumes -f            |
| Inspect WM Class       | Alt+F2 -> lg -> Windows                        |
| Increase inotify       | sudo sysctl -p /etc/sysctl.d/99-inotify.conf   |
| Reload Desktop DB      | update-desktop-database ~/.local/share/appl... |
+-------------------------------------------------------------------------+
```

## INDEX

- 01. [MOBILE RUNTIME](#01-mobile-runtime)
- 02. [CONTAINERS & DOCKER ENGINE](#02-containers--docker-engine)
- 03. [GIT VCS & SECURITY](#03-git-vcs--security)
- 04. [LINUX DESKTOP & SHELL](#04-linux-desktop--shell)
- 05. [HARDWARE & SYSTEM MODS](#05-hardware--system-mods)
- 06. [KERNEL & DRIVER TROUBLESHOOTING](#06-kernel--driver-troubleshooting)

---

## 01. MOBILE RUNTIME

### Environment Variables
```bash
export ANDROID_HOME=$HOME/Android/Sdk
export PATH=$PATH:$ANDROID_HOME/tools:$ANDROID_HOME/platform-tools
```

### Memory Management Alias
```bash
alias lp='sudo sync && echo 3 | sudo tee /proc/sys/vm/drop_caches'
```

### Android Emulator GPU Freeze Fix
1. Alter window timeout:
```bash
gsettings set org.gnome.mutter check-alive-timeout 0
```
2. Set host GPU in `~/.android/avd/<avd_name>.avd/config.ini`:
```ini
hw.gpu.mode=host
```
3. Grant KVM rights:
```bash
sudo usermod -aG kvm $USER
groups
```

### Java Toolchain Setup
```bash
sudo apt install openjdk-17-jdk
echo "export JAVA_HOME=/usr/lib/jvm/java-17-openjdk-amd64" >> ~/.zshrc
echo "export PATH=\$JAVA_HOME/bin:\$PATH" >> ~/.zshrc
source ~/.zshrc
```

---

## 02. CONTAINERS & DOCKER ENGINE

### Nuclear Cache Cleanup
```bash
docker rm -f $(docker ps -aq) 2>/dev/null; docker rmi -f $(docker images -aq) 2>/dev/null; docker volume rm $(docker volume ls -q) 2>/dev/null && docker network prune -f && docker system prune -a --volumes -f && docker system df
```

### External Storage Migration (dockerd + containerd)

#### 1. Disk Mountpoint (`/etc/fstab`)
```text
UUID=aa377c86-d063-47d9-b91d-9ad912d09fab /run/media/data/hd-externo-1tb ext4 defaults,nofail,x-systemd.automount 0 2
```
```bash
sudo mkdir -p /run/media/data/hd-externo-1tb
sudo chmod 755 /run/media/data/
sudo chown $USER:$USER /run/media/data/hd-externo-1tb
sudo systemctl daemon-reload && sudo mount -a
```

#### 2. Stop Daemons
```bash
sudo systemctl stop docker docker.socket containerd
```

#### 3. Daemon Configurations

`/etc/docker/daemon.json`:
```json
{
  "data-root": "/run/media/data/hd-externo-1tb/docker",
  "runtimes": {
    "nvidia": {
      "args": [],
      "path": "nvidia-container-runtime"
    }
  }
}
```

`/etc/containerd/config.toml`:
```toml
root = "/run/media/data/hd-externo-1tb/containerd"
```

#### 4. Data Migration & Start
```bash
sudo rsync -aP /var/lib/docker/ /run/media/data/hd-externo-1tb/docker/
sudo rsync -aP /var/lib/containerd/ /run/media/data/hd-externo-1tb/containerd/
sudo systemctl start containerd && sudo systemctl start docker
docker info | grep "Docker Root Dir"
sudo rm -rf /var/lib/docker /var/lib/containerd
```

---

## 03. GIT VCS & SECURITY

### SSH Keygen (ED25519)
```bash
ssh-keygen -t ed25519 -C "email@gmail.com" -f ~/.ssh/id_ed25519 -N "" && git config --global user.email "email@gmail.com" && git config --global user.name "Mateus Santos" && cat ~/.ssh/id_ed25519.pub
```

### Purge Large Binaries from History
```bash
git filter-branch --force --index-filter "git rm --cached --ignore-unmatch *.apk *.aab *.img *.jpg *.pdf *.mp4" --prune-empty --tag-name-filter cat -- --all
git reflog expire --expire=now --all
git gc --prune=now --aggressive
git push origin --force --all
```

---

## 04. LINUX DESKTOP & SHELL

### Oh My Zsh + Autosuggestions
```bash
sh -c "$(curl -fsSL https://raw.githubusercontent.com/ohmyzsh/ohmyzsh/master/tools/install.sh)"
git clone https://github.com/zsh-users/zsh-autosuggestions ${ZSH_CUSTOM:-~/.oh-my-zsh/custom}/plugins/zsh-autosuggestions && sed -i 's/plugins=(git)/plugins=(git zsh-autosuggestions)/' ~/.zshrc && source ~/.zshrc
chsh -s $(which zsh)
```

### Target Runlevels
```bash
sudo systemctl set-default multi-user.target
sudo systemctl set-default graphical.target
```

### Desktop Launcher Template (`~/.local/share/applications/app.desktop`)
```ini
[Desktop Entry]
Name=CustomApp
Type=Application
Exec=/path/to/binary %U
Icon=/path/to/icon.png
Terminal=false
StartupWMClass=window-wm-class
Categories=Development;
```
```bash
update-desktop-database ~/.local/share/applications/
```

### Window Stacking
```bash
sudo apt install wmctrl
wmctrl -r :SELECT: -b add,above
wmctrl -r :SELECT: -b remove,above
```

---

## 05. HARDWARE & SYSTEM MODS

### Secondary Drives (`/etc/fstab`)
```text
UUID=your_UUID /run/media/data/hd-externo-1tb ext4 defaults,nofail 0 2
```

### DualShock 4 Engine Config

`/etc/bluetooth/input.conf`:
```ini
ClassicBondedOnly=false
```

`/etc/bluetooth/main.conf`:
```ini
[Policy]
AutoEnable=true
ReconnectAttempts=7
ReconnectIntervals=1,2,4,8,16,32,64
ReconnectUUIDs=00001112-0000-1000-8000-00805f9b34fb,0000111f-0000-1000-8000-00805f9b34fb,0000110a-0000-1000-8000-00805f9b34fb,0000110b-0000-1000-8000-00805f9b34fb
```
```bash
sudo systemctl restart bluetooth
ls /dev/input/js*
```

### USB Polling Rate 1000Hz Mod
```bash
curl -Lo /tmp/usb-oc-dkms.deb https://github.com/p0358/usb_oc-dkms/releases/download/v1.1/usb-oc-dkms_1.1_amd64.deb
sudo apt install -y /tmp/usb-oc-dkms.deb
echo "options usb_oc interrupt_interval_override=054c:09cc:1" | sudo tee /etc/modprobe.d/usb_oc.conf
echo "usb_oc" | sudo tee /etc/modules-load.d/usb_oc.conf
sudo modprobe usb_oc
```

---

## 06. KERNEL & DRIVER TROUBLESHOOTING

### Wayland Dual GPU Freeze
```bash
echo "blacklist amdgpu" | sudo tee /etc/modprobe.d/blacklist-amdgpu.conf
echo "options nvidia-drm modeset=1" | sudo tee /etc/modprobe.d/nvidia-drm.conf
sudo update-initramfs -u
sudo reboot
```

### NVIDIA Modeset Timeout (Steam / Games)
In `/etc/default/grub`:
```text
GRUB_CMDLINE_LINUX_DEFAULT="quiet splash nvidia-drm.modeset=1 nvidia.NVreg_PreserveVideoMemoryAllocations=1 nvidia.NVreg_EnableGpuFirmware=0"
```
```bash
sudo update-grub && sudo reboot
```

### Inotify Watcher Overflow (ENOSPC)
In `/etc/sysctl.d/99-inotify.conf`:
```ini
fs.inotify.max_user_watches=524288
fs.inotify.max_user_instances=1024
fs.inotify.max_queued_events=32768
```
```bash
sudo sysctl --system
```

### Dock Gear Icon Deduplication (StartupWMClass)
1. Query window class via Looking Glass (`Alt + F2` -> `lg` -> `Windows`).
2. Add `StartupWMClass=<class>` inside `~/.local/share/applications/pinned-app.desktop`.
3. Refresh database:
```bash
update-desktop-database ~/.local/share/applications/
```
