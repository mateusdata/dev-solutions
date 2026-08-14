# Linux & Dev Handbook (Interactive Edition)

> Clique nas seções abaixo para expandir e recolher os módulos de configuração e troubleshooting.

---

<details open>
<summary><h3>1. Mobile & Android SDK</h3></summary>

#### Configuração de Variáveis de Ambiente
```bash
export ANDROID_HOME=$HOME/Android/Sdk
export PATH=$PATH:$ANDROID_HOME/tools:$ANDROID_HOME/platform-tools
```

#### Alias para Limpeza de Memória RAM
```bash
alias lp='sudo sync && echo 3 | sudo tee /proc/sys/vm/drop_caches'
```

#### Troubleshooting: Emulador Travando ("Not Responding")
```bash
gsettings set org.gnome.mutter check-alive-timeout 0
```
No arquivo `~/.android/avd/NOME_AVD.avd/config.ini`:
```ini
hw.gpu.mode=host
```
Permissões KVM:
```bash
sudo usermod -aG kvm $USER
groups
```

#### Troubleshooting: JAVA_HOME Não Configurado
```bash
sudo apt install openjdk-17-jdk
echo "export JAVA_HOME=/usr/lib/jvm/java-17-openjdk-amd64" >> ~/.zshrc
echo "export PATH=\$JAVA_HOME/bin:\$PATH" >> ~/.zshrc
source ~/.zshrc
```

</details>

---

<details>
<summary><h3>2. Docker & Containerd Storage</h3></summary>

#### Limpeza Total de Cache
```bash
docker rm -f $(docker ps -aq) 2>/dev/null; docker rmi -f $(docker images -aq) 2>/dev/null; docker volume rm $(docker volume ls -q) 2>/dev/null && docker network prune -f && docker system prune -a --volumes -f && docker system df
```

#### Migração para HD Externo

1. **Montagem (`/etc/fstab`):**
   ```text
   UUID=aa377c86-d063-47d9-b91d-9ad912d09fab /run/media/data/hd-externo-1tb ext4 defaults,nofail,x-systemd.automount 0 2
   ```
   ```bash
   sudo mkdir -p /run/media/data/hd-externo-1tb
   sudo chmod 755 /run/media/data/
   sudo chown $USER:$USER /run/media/data/hd-externo-1tb
   sudo systemctl daemon-reload && sudo mount -a
   ```

2. **Parar Serviços:**
   ```bash
   sudo systemctl stop docker docker.socket containerd
   ```

3. **Configuração do Docker (`/etc/docker/daemon.json`):**
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

4. **Configuração do Containerd (`/etc/containerd/config.toml`):**
   ```toml
   root = "/run/media/data/hd-externo-1tb/containerd"
   ```

5. **Transferência e Inicialização:**
   ```bash
   sudo rsync -aP /var/lib/docker/ /run/media/data/hd-externo-1tb/docker/
   sudo rsync -aP /var/lib/containerd/ /run/media/data/hd-externo-1tb/containerd/
   sudo systemctl start containerd && sudo systemctl start docker
   docker info | grep "Docker Root Dir"
   sudo rm -rf /var/lib/docker /var/lib/containerd
   ```

</details>

---

<details>
<summary><h3>3. Git & Chaves SSH</h3></summary>

#### Geração de Chave SSH ED25519
```bash
ssh-keygen -t ed25519 -C "email@gmail.com" -f ~/.ssh/id_ed25519 -N "" && git config --global user.email "email@gmail.com" && git config --global user.name "Mateus Santos" && cat ~/.ssh/id_ed25519.pub
```

#### Purga de Arquivos Grandes do Histórico
```bash
git filter-branch --force --index-filter "git rm --cached --ignore-unmatch *.apk *.aab *.img *.jpg *.pdf *.mp4" --prune-empty --tag-name-filter cat -- --all
git reflog expire --expire=now --all
git gc --prune=now --aggressive
git push origin --force --all
```

</details>

---

<details>
<summary><h3>4. Sistema Linux, Shell & Lançadores</h3></summary>

#### Oh My Zsh + Autosuggestions
```bash
sh -c "$(curl -fsSL https://raw.githubusercontent.com/ohmyzsh/ohmyzsh/master/tools/install.sh)"
git clone https://github.com/zsh-users/zsh-autosuggestions ${ZSH_CUSTOM:-~/.oh-my-zsh/custom}/plugins/zsh-autosuggestions && sed -i 's/plugins=(git)/plugins=(git zsh-autosuggestions)/' ~/.zshrc && source ~/.zshrc
chsh -s $(which zsh)
```

#### Controle de Runlevel Gráfico / Texto
```bash
sudo systemctl set-default multi-user.target
sudo systemctl set-default graphical.target
```

#### Gravação de Imagens ISO
```bash
flatpak install flathub org.fedoraproject.MediaWriter
```

#### Pinagem de Janela no Topo
```bash
sudo apt install wmctrl
wmctrl -r :SELECT: -b add,above
wmctrl -r :SELECT: -b remove,above
```

#### Configuração de Lançadores (.desktop) & PinApp
Arquivo `~/.local/share/applications/app.desktop`:
```ini
[Desktop Entry]
Name=NomeDoApp
Type=Application
Exec=/caminho/executavel %U
Icon=/caminho/icone.png
Terminal=false
StartupWMClass=classe-da-janela
Categories=Development;
```
```bash
update-desktop-database ~/.local/share/applications/
```

</details>

---

<details>
<summary><h3>5. Hardware, Bluetooth & Controles</h3></summary>

#### Montagem Automática (`/etc/fstab`)
```text
UUID=your_UUID /run/media/data/hd-externo-1tb ext4 defaults,nofail 0 2
```

#### Conexão do DualShock 4 no Linux Mint

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

#### Teste de Polling Rate
```bash
git clone https://github.com/cakama3a/Polling.git ~/Downloads/Polling
cd ~/Downloads/Polling && uv run python Python.py
```

#### Overclock USB 1000Hz
```bash
curl -Lo /tmp/usb-oc-dkms.deb https://github.com/p0358/usb_oc-dkms/releases/download/v1.1/usb-oc-dkms_1.1_amd64.deb
sudo apt install -y /tmp/usb-oc-dkms.deb
echo "options usb_oc interrupt_interval_override=054c:09cc:1" | sudo tee /etc/modprobe.d/usb_oc.conf
echo "usb_oc" | sudo tee /etc/modules-load.d/usb_oc.conf
sudo modprobe usb_oc
```

</details>

---

<details>
<summary><h3>6. Solução de Problemas do Linux (Troubleshooting)</h3></summary>

#### Tela Congelando (Dual GPU NVIDIA + AMD no Wayland)
```bash
echo "blacklist amdgpu" | sudo tee /etc/modprobe.d/blacklist-amdgpu.conf
echo "options nvidia-drm modeset=1" | sudo tee /etc/modprobe.d/nvidia-drm.conf
sudo update-initramfs -u
sudo reboot
```

#### Tela Preta em Jogos (GPU Timeout no Wayland)
Em `/etc/default/grub`:
```text
GRUB_CMDLINE_LINUX_DEFAULT="quiet splash nvidia-drm.modeset=1 nvidia.NVreg_PreserveVideoMemoryAllocations=1 nvidia.NVreg_EnableGpuFirmware=0"
```
```bash
sudo update-grub && sudo reboot
```

#### Restauração do PATH Quebrado
```bash
export PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:$PATH
/usr/bin/sudo /usr/bin/apt install ubuntu-desktop gnome-session gnome-shell
```

#### Limite de File Watchers do Inotify (ENOSPC)
Em `/etc/sysctl.d/99-inotify.conf`:
```ini
fs.inotify.max_user_watches=524288
fs.inotify.max_user_instances=1024
fs.inotify.max_queued_events=32768
```
```bash
sudo sysctl --system
```

#### Ícone de Engrenagem Duplicado na Dock (StartupWMClass)
1. Descubra a classe via Looking Glass: `Alt + F2` ➔ `lg` ➔ aba **Windows**.
2. Adicione `StartupWMClass=classe` no arquivo `.desktop` em `~/.local/share/applications/`.
3. Recarregue:
```bash
update-desktop-database ~/.local/share/applications/
```

</details>
