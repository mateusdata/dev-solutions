# DEV-SOLUTIONS // SYSTEM SPEC SHEET

> Guia de engenharia de sistemas, configurações de baixo nível e recuperação de falhas operacionais para Linux.

---

## SUMÁRIO OPERACIONAL

```
+-------------------------------------------------------------------------------+
| CORE PROFILES                                                                 |
+-------------------+-----------------------------------------------------------+
| 01. MOBILE STACK  | Android SDK, AVD GPU Acceleration, Java OpenJDK 17        |
| 02. CONTAINERS    | Docker Data Root Partitioning, Containerd Storage         |
| 03. SECURE VCS    | ED25519 Cryptography, Git Tree Rewriting & Pruning        |
| 04. SHELL & UI    | Oh My Zsh Engine, Desktop Entries, System Window Binding  |
| 05. HARDWARE IO   | Fstab Automount, Bluetooth Stack, USB Polling Overclock   |
| 06. KERNEL FIXES  | NVIDIA DRM Modeset, Inotify Limits, Wayland GPU Isolation|
+-------------------+-----------------------------------------------------------+
```

---

## 1. MOBILE DEVELOPMENT & RUNTIME

> [!TIP]
> A aceleração por hardware direta no emulador Android elimina gargalos de CPU e quedas de FPS no desenvolvimento móvel.

### Variáveis Globais (`~/.zshrc`)
```bash
export ANDROID_HOME=$HOME/Android/Sdk
export PATH=$PATH:$ANDROID_HOME/tools:$ANDROID_HOME/platform-tools
export JAVA_HOME=/usr/lib/jvm/java-17-openjdk-amd64
export PATH=$JAVA_HOME/bin:$PATH
```

### Limpeza Instantânea de Memória
```bash
alias lp='sudo sync && echo 3 | sudo tee /proc/sys/vm/drop_caches'
```

### Otimização Crítica do AVD
```bash
gsettings set org.gnome.mutter check-alive-timeout 0
sed -i 's/hw.gpu.mode=.*/hw.gpu.mode=host/' ~/.android/avd/*.avd/config.ini
sudo usermod -aG kvm $USER
```

---

## 2. CONTAINER STORAGE ARCHITECTURE

> [!IMPORTANT]
> Ao mover o armazenamento do Docker para um disco secundário, ambos os daemons (`dockerd` e `containerd`) precisam ser reconfigurados para evitar gravação oculta no SSD principal.

### Particionamento e Montagem Automática (`/etc/fstab`)
```text
UUID=aa377c86-d063-47d9-b91d-9ad912d09fab /run/media/data/hd-externo-1tb ext4 defaults,nofail,x-systemd.automount 0 2
```

### Pipeline de Migração
```bash
sudo systemctl stop docker docker.socket containerd
```

#### `/etc/docker/daemon.json`
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

#### `/etc/containerd/config.toml`
```toml
root = "/run/media/data/hd-externo-1tb/containerd"
```

```bash
sudo rsync -aP /var/lib/docker/ /run/media/data/hd-externo-1tb/docker/
sudo rsync -aP /var/lib/containerd/ /run/media/data/hd-externo-1tb/containerd/
sudo systemctl start containerd && sudo systemctl start docker
sudo rm -rf /var/lib/docker /var/lib/containerd
```

---

## 3. GIT SECURITY & HISTORY AUDITING

### Geração de Par de Chaves ED25519
```bash
ssh-keygen -t ed25519 -C "email@gmail.com" -f ~/.ssh/id_ed25519 -N "" && git config --global user.email "email@gmail.com" && git config --global user.name "Mateus Santos" && cat ~/.ssh/id_ed25519.pub
```

### Limpeza de Objetos Binários do Git
```bash
git filter-branch --force --index-filter "git rm --cached --ignore-unmatch *.apk *.aab *.img *.jpg *.pdf *.mp4" --prune-empty --tag-name-filter cat -- --all
git reflog expire --expire=now --all
git gc --prune=now --aggressive
git push origin --force --all
```

---

## 4. SHELL ENVIRONMENT & DESKTOP ENTRIES

### Setup da Shell Zsh
```bash
sh -c "$(curl -fsSL https://raw.githubusercontent.com/ohmyzsh/ohmyzsh/master/tools/install.sh)"
git clone https://github.com/zsh-users/zsh-autosuggestions ${ZSH_CUSTOM:-~/.oh-my-zsh/custom}/plugins/zsh-autosuggestions && sed -i 's/plugins=(git)/plugins=(git zsh-autosuggestions)/' ~/.zshrc && source ~/.zshrc
chsh -s $(which zsh)
```

### Especificação de Lançador (`~/.local/share/applications/`)
```ini
[Desktop Entry]
Name=NomeDoApp
Type=Application
Exec=/caminho/do/executavel %U
Icon=/caminho/do/icone.png
Terminal=false
StartupWMClass=classe-da-janela
Categories=Development;
```
```bash
update-desktop-database ~/.local/share/applications/
```

---

## 5. HARDWARE TUNING & HID OVERCLOCK

### Configuração do Protocolo Bluetooth (DualShock 4)
No arquivo `/etc/bluetooth/input.conf`:
```ini
ClassicBondedOnly=false
```
No arquivo `/etc/bluetooth/main.conf`:
```ini
[Policy]
AutoEnable=true
ReconnectAttempts=7
ReconnectIntervals=1,2,4,8,16,32,64
ReconnectUUIDs=00001112-0000-1000-8000-00805f9b34fb,0000111f-0000-1000-8000-00805f9b34fb,0000110a-0000-1000-8000-00805f9b34fb,0000110b-0000-1000-8000-00805f9b34fb
```
```bash
sudo systemctl restart bluetooth
```

### Overclock de Taxa de Polling USB (1000Hz / 1ms)
```bash
curl -Lo /tmp/usb-oc-dkms.deb https://github.com/p0358/usb_oc-dkms/releases/download/v1.1/usb-oc-dkms_1.1_amd64.deb
sudo apt install -y /tmp/usb-oc-dkms.deb
echo "options usb_oc interrupt_interval_override=054c:09cc:1" | sudo tee /etc/modprobe.d/usb_oc.conf
echo "usb_oc" | sudo tee /etc/modules-load.d/usb_oc.conf
sudo modprobe usb_oc
```

---

## 6. KERNEL & DISPLAY SERVER TROUBLESHOOTING

### Conflito Dual GPU no Wayland (NVIDIA + AMD)
```bash
echo "blacklist amdgpu" | sudo tee /etc/modprobe.d/blacklist-amdgpu.conf
echo "options nvidia-drm modeset=1" | sudo tee /etc/modprobe.d/nvidia-drm.conf
sudo update-initramfs -u && sudo reboot
```

### Prevenção de GPU Timeout em Jogos (GRUB)
Em `/etc/default/grub`:
```text
GRUB_CMDLINE_LINUX_DEFAULT="quiet splash nvidia-drm.modeset=1 nvidia.NVreg_PreserveVideoMemoryAllocations=1 nvidia.NVreg_EnableGpuFirmware=0"
```
```bash
sudo update-grub && sudo reboot
```

### Exaustão de File Watchers (`ENOSPC`)
Em `/etc/sysctl.d/99-inotify.conf`:
```ini
fs.inotify.max_user_watches=524288
fs.inotify.max_user_instances=1024
fs.inotify.max_queued_events=32768
```
```bash
sudo sysctl --system
```

### Desduplicação de Ícones na Dock (StartupWMClass)
1. Pressione `Alt + F2`, digite `lg` e abra a aba **Windows** do Looking Glass.
2. Copie o valor de `wmclass` do programa em execução.
3. Insira `StartupWMClass=<valor>` no `.desktop` do app em `~/.local/share/applications/`.
4. Execute:
```bash
update-desktop-database ~/.local/share/applications/
```
