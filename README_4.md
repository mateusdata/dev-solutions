# Dev Solutions Knowledge Base

**Document Version:** 2.4.0  
**Environment:** Linux Mint / Ubuntu (x86_64)  
**Kernel Targets:** 6.x LTS  

---

## Estrutura Modular da Base de Conhecimento

```
dev-solutions/
├── 01-mobile/       -> SDKs, Emuladores, Otimizações de Memória
├── 02-containers/   -> Docker Engine, Armazenamento Externo, Daemon
├── 03-vcs-git/      -> Chaves Criptográficas, Higienização de Repositório
├── 04-desktop-os/   -> Zsh Shell, Lançadores .desktop, Gerenciamento de Janelas
├── 05-hardware/     -> Armazenamento Permanente, Conexões Bluetooth, HID Overclock
└── 06-diagnostics/  -> Conflitos de GPU, Kernel Crash Recovery, System Watchers
```

---

## Modulo 01: Desenvolvimento Mobile

### 1.1 Variáveis de Ambiente Android
Arquivo alvo: `~/.zshrc`

```bash
export ANDROID_HOME=$HOME/Android/Sdk
export PATH=$PATH:$ANDROID_HOME/tools:$ANDROID_HOME/platform-tools
```

### 1.2 Limpeza Forçada de Cache de Memória
```bash
alias lp='sudo sync && echo 3 | sudo tee /proc/sys/vm/drop_caches'
```

### 1.3 Troubleshooting: Estabilidade do Emulador AVD

| Componente | Configuração Padrão | Ajuste Recomendado | Efeito |
| :--- | :--- | :--- | :--- |
| **GPU Rendering** | `auto` | `host` | Força renderização direta pela GPU física |
| **Timeout GNOME** | `5000` ms | `0` (desativado) | Previne pop-up "Not Responding" |
| **Aceleração KVM** | Usuário padrão | Membro do grupo `kvm` | Acesso de hardware ao `/dev/kvm` |

Procedimento de aplicação:
```bash
gsettings set org.gnome.mutter check-alive-timeout 0
sed -i 's/hw.gpu.mode=.*/hw.gpu.mode=host/' ~/.android/avd/*.avd/config.ini
sudo usermod -aG kvm $USER
```

---

## Modulo 02: Infraestrutura Docker & Armazenamento

### 2.1 Higienização do Subsistema Docker
```bash
docker rm -f $(docker ps -aq) 2>/dev/null; docker rmi -f $(docker images -aq) 2>/dev/null; docker volume rm $(docker volume ls -q) 2>/dev/null && docker network prune -f && docker system prune -a --volumes -f && docker system df
```

### 2.2 Reatribuição de Diretório Raiz (`dockerd` e `containerd`)

#### Passo 1: Ponto de Montagem Permanente (`/etc/fstab`)
```text
UUID=aa377c86-d063-47d9-b91d-9ad912d09fab /run/media/data/hd-externo-1tb ext4 defaults,nofail,x-systemd.automount 0 2
```
```bash
sudo mkdir -p /run/media/data/hd-externo-1tb
sudo chmod 755 /run/media/data/
sudo chown $USER:$USER /run/media/data/hd-externo-1tb
sudo systemctl daemon-reload && sudo mount -a
```

#### Passo 2: Configuração dos Serviços
Pare os daemons:
```bash
sudo systemctl stop docker docker.socket containerd
```

Arquivo `/etc/docker/daemon.json`:
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

Arquivo `/etc/containerd/config.toml`:
```toml
root = "/run/media/data/hd-externo-1tb/containerd"
```

#### Passo 3: Migração e Validação
```bash
sudo rsync -aP /var/lib/docker/ /run/media/data/hd-externo-1tb/docker/
sudo rsync -aP /var/lib/containerd/ /run/media/data/hd-externo-1tb/containerd/
sudo systemctl start containerd && sudo systemctl start docker
docker info | grep "Docker Root Dir"
sudo rm -rf /var/lib/docker /var/lib/containerd
```

---

## Modulo 03: Git & Controle Criptográfico

### 3.1 Identidade SSH ED25519
```bash
ssh-keygen -t ed25519 -C "email@gmail.com" -f ~/.ssh/id_ed25519 -N "" && git config --global user.email "email@gmail.com" && git config --global user.name "Mateus Santos" && cat ~/.ssh/id_ed25519.pub
```

### 3.2 Purga de Binários do Histórico Git
```bash
git filter-branch --force --index-filter "git rm --cached --ignore-unmatch *.apk *.aab *.img *.jpg *.pdf *.mp4" --prune-empty --tag-name-filter cat -- --all
git reflog expire --expire=now --all
git gc --prune=now --aggressive
git push origin --force --all
```

---

## Modulo 04: Sistema Operacional & Lançadores Desktop

### 4.1 Ambiente de Terminal Zsh
```bash
sh -c "$(curl -fsSL https://raw.githubusercontent.com/ohmyzsh/ohmyzsh/master/tools/install.sh)"
git clone https://github.com/zsh-users/zsh-autosuggestions ${ZSH_CUSTOM:-~/.oh-my-zsh/custom}/plugins/zsh-autosuggestions && sed -i 's/plugins=(git)/plugins=(git zsh-autosuggestions)/' ~/.zshrc && source ~/.zshrc
chsh -s $(which zsh)
```

### 4.2 Lançadores Personalizados (.desktop)
Localização: `~/.local/share/applications/`

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

Atualização do cache do ambiente:
```bash
update-desktop-database ~/.local/share/applications/
```

---

## Modulo 05: Hardware, Bluetooth & HID Overclock

### 5.1 Configuração do Controlador DualShock 4

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

### 5.2 Polling Rate e Overclock USB (1000Hz)
```bash
curl -Lo /tmp/usb-oc-dkms.deb https://github.com/p0358/usb_oc-dkms/releases/download/v1.1/usb-oc-dkms_1.1_amd64.deb
sudo apt install -y /tmp/usb-oc-dkms.deb
echo "options usb_oc interrupt_interval_override=054c:09cc:1" | sudo tee /etc/modprobe.d/usb_oc.conf
echo "usb_oc" | sudo tee /etc/modules-load.d/usb_oc.conf
sudo modprobe usb_oc
```

---

## Modulo 06: Matriz de Diagnósticos & Correções

### 6.1 Conflito Dual GPU (NVIDIA + AMD) no Wayland
```bash
echo "blacklist amdgpu" | sudo tee /etc/modprobe.d/blacklist-amdgpu.conf
echo "options nvidia-drm modeset=1" | sudo tee /etc/modprobe.d/nvidia-drm.conf
sudo update-initramfs -u && sudo reboot
```

### 6.2 Prevenção de GPU Timeout em Jogos
Em `/etc/default/grub`:
```text
GRUB_CMDLINE_LINUX_DEFAULT="quiet splash nvidia-drm.modeset=1 nvidia.NVreg_PreserveVideoMemoryAllocations=1 nvidia.NVreg_EnableGpuFirmware=0"
```
```bash
sudo update-grub && sudo reboot
```

### 6.3 Ajuste de Limite de File Watchers (inotify)
Em `/etc/sysctl.d/99-inotify.conf`:
```ini
fs.inotify.max_user_watches=524288
fs.inotify.max_user_instances=1024
fs.inotify.max_queued_events=32768
```
```bash
sudo sysctl --system
```

### 6.4 Correção de Ícone Duplicado de Engrenagem na Dock
1. Abra o Looking Glass: `Alt + F2` ➔ `lg` ➔ aba **Windows**.
2. Identifique o valor do campo `wmclass` do programa.
3. Adicione `StartupWMClass=classe` ao arquivo `.desktop` correspondente.
4. Execute:
```bash
update-desktop-database ~/.local/share/applications/
```
