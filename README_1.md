# Linux & Dev Handbook

> Manual unificado de configuração de ambiente, otimização de sistema e resolução de problemas.

---

## Painel de Navegação Rápida

| Categoria | Tópicos de Setup | Solução de Problemas |
| :--- | :--- | :--- |
| **Mobile** | Android SDK, Alias de RAM | Emulador travando, JAVA_HOME |
| **Docker** | Limpeza de Cache, Armazenamento em HD Externo | Configuração de Daemon & Containerd |
| **Git** | Chaves SSH ED25519, Limpeza de Histórico | Filtro de arquivos pesados |
| **Linux & Desktop** | Oh My Zsh, Modo Texto, Lançadores Desktop | Tela congelando (Wayland), Ícones duplicados |
| **Hardware** | Montagem Automática (fstab), Hashcat | Timeout de GPU NVIDIA, File Watchers (inotify) |
| **Controles** | DualShock 4, Polling Rate | Overclock USB 1000Hz |

---

## 1. Mobile Development

### Configuração do Android SDK no Zsh
Adicione as variáveis de ambiente ao arquivo `~/.zshrc`:

```bash
export ANDROID_HOME=$HOME/Android/Sdk
export PATH=$PATH:$ANDROID_HOME/tools:$ANDROID_HOME/platform-tools
```

### Alias para Liberação Imediata de Memória RAM
Adicione ao `~/.zshrc`:

```bash
alias lp='sudo sync && echo 3 | sudo tee /proc/sys/vm/drop_caches'
```

---

## 2. Docker & Containerd

### Limpeza Completa de Cache e Volumes Órfãos
```bash
docker rm -f $(docker ps -aq) 2>/dev/null; docker rmi -f $(docker images -aq) 2>/dev/null; docker volume rm $(docker volume ls -q) 2>/dev/null && docker network prune -f && docker system prune -a --volumes -f && docker system df
```

### Migração do Docker e Containerd para HD Externo

> [!IMPORTANT]
> O Docker depende de dois serviços independentes: **dockerd** e **containerd**. Ambos devem ser apontados para o disco secundário.

#### 1. Montagem do HD no `/etc/fstab`
```bash
sudo mkdir -p /run/media/data/hd-externo-1tb
sudo nano /etc/fstab
```

Insira a linha:
```text
UUID=aa377c86-d063-47d9-b91d-9ad912d09fab /run/media/data/hd-externo-1tb ext4 defaults,nofail,x-systemd.automount 0 2
```

Ajuste de permissões e teste de montagem:
```bash
sudo chmod 755 /run/media/data/
sudo chown $USER:$USER /run/media/data/hd-externo-1tb
sudo systemctl daemon-reload
sudo mount -a
```

#### 2. Parada dos Serviços
```bash
sudo systemctl stop docker docker.socket containerd
```

#### 3. Configuração do Daemon Docker (`/etc/docker/daemon.json`)
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

#### 4. Configuração do Containerd (`/etc/containerd/config.toml`)
```bash
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml
sudo nano /etc/containerd/config.toml
```

Altere o campo `root`:
```toml
root = "/run/media/data/hd-externo-1tb/containerd"
```

#### 5. Sincronização e Inicialização
```bash
sudo rsync -aP /var/lib/docker/ /run/media/data/hd-externo-1tb/docker/
sudo rsync -aP /var/lib/containerd/ /run/media/data/hd-externo-1tb/containerd/
sudo systemctl start containerd && sudo systemctl start docker
```

Verificação:
```bash
docker info | grep "Docker Root Dir"
```

Remoção das pastas antigas do SSD:
```bash
sudo rm -rf /var/lib/docker
sudo rm -rf /var/lib/containerd
```

---

## 3. Git & Controle de Versão

### Geração e Configuração de Chave SSH ED25519
```bash
ssh-keygen -t ed25519 -C "email@gmail.com" -f ~/.ssh/id_ed25519 -N "" && git config --global user.email "email@gmail.com" && git config --global user.name "Mateus Santos" && cat ~/.ssh/id_ed25519.pub
```

### Limpeza Permanente de Arquivos Pesados do Histórico Git
```bash
git filter-branch --force --index-filter "git rm --cached --ignore-unmatch *.apk *.aab *.img *.jpg *.pdf *.mp4" --prune-empty --tag-name-filter cat -- --all
git reflog expire --expire=now --all
git gc --prune=now --aggressive
git push origin --force --all
```

---

## 4. Ambiente Linux & Produtividade

### Oh My Zsh & Autosuggestions
Instalação completa:
```bash
sh -c "$(curl -fsSL https://raw.githubusercontent.com/ohmyzsh/ohmyzsh/master/tools/install.sh)"
git clone https://github.com/zsh-users/zsh-autosuggestions ${ZSH_CUSTOM:-~/.oh-my-zsh/custom}/plugins/zsh-autosuggestions && sed -i 's/plugins=(git)/plugins=(git zsh-autosuggestions)/' ~/.zshrc && source ~/.zshrc
```

Alterar shell padrão para Zsh:
```bash
chsh -s $(which zsh)
sudo chsh -s $(which zsh) root
```

### SSH & Montagem de Diretório Remoto
```bash
sudo apt install openssh-server sshfs
sshfs user@servidor:/caminho/remoto ./pasta_local
```

### Controle de Modo Gráfico / Modo Texto
```bash
sudo systemctl set-default multi-user.target
sudo systemctl set-default graphical.target
sudo systemctl isolate multi-user.target
```

### Gravação de Pendrive Bootável
```bash
flatpak install flathub org.fedoraproject.MediaWriter
```

### Gerenciamento de Janela no Topo
```bash
sudo apt install wmctrl
wmctrl -r :SELECT: -b add,above
wmctrl -r :SELECT: -b remove,above
```

### Lançadores Desktop & PinApp
Estrutura padrão de inicializador em `~/.local/share/applications/nome-do-app.desktop`:

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

Atualização da base do sistema:
```bash
update-desktop-database ~/.local/share/applications/
```

---

## 5. Hardware, Armazenamento & Periféricos

### Montagem Automática de Discos Secundários
Descobrir UUID:
```bash
sudo blkid
```

Adicionar ao `/etc/fstab`:
```text
UUID=your_UUID /run/media/data/hd-externo-1tb ext4 defaults,nofail 0 2
```

### Hashcat Benchmark / Execução
```bash
hashcat -m 22000 arquivo.hc22000 -a 3 ?d?d?d?d?d?d?d?d
```

### Pareamento e Conexão DualShock 4
Edite `/etc/bluetooth/input.conf`:
```ini
ClassicBondedOnly=false
```

Edite `/etc/bluetooth/main.conf`:
```ini
[Policy]
AutoEnable=true
ReconnectAttempts=7
ReconnectIntervals=1,2,4,8,16,32,64
ReconnectUUIDs=00001112-0000-1000-8000-00805f9b34fb,0000111f-0000-1000-8000-00805f9b34fb,0000110a-0000-1000-8000-00805f9b34fb,0000110b-0000-1000-8000-00805f9b34fb
```

Reinicialização do serviço:
```bash
sudo systemctl restart bluetooth
ls /dev/input/js*
```

### Teste de Polling Rate (Gamepadla)
```bash
git clone https://github.com/cakama3a/Polling.git ~/Downloads/Polling
cd ~/Downloads/Polling && uv run python Python.py
```

### Overclock de Controle USB (1000Hz)
```bash
curl -Lo /tmp/usb-oc-dkms.deb https://github.com/p0358/usb_oc-dkms/releases/download/v1.1/usb-oc-dkms_1.1_amd64.deb
sudo apt install -y /tmp/usb-oc-dkms.deb
echo "options usb_oc interrupt_interval_override=054c:09cc:1" | sudo tee /etc/modprobe.d/usb_oc.conf
echo "usb_oc" | sudo tee /etc/modules-load.d/usb_oc.conf
sudo modprobe usb_oc
```

---

## 6. Guia de Troubleshooting

### Emulador Android Travando com "Emulator is Not Responding"
* **Causa:** `hw.gpu.mode=auto` oscilando entre renderização por software e GPU dedicada.
* **Ajuste no GNOME:**
  ```bash
  gsettings set org.gnome.mutter check-alive-timeout 0
  ```
* **Ajuste no AVD (`~/.android/avd/NOME_AVD.avd/config.ini`):**
  ```ini
  hw.gpu.mode=host
  ```
* **Permissões KVM:**
  ```bash
  sudo usermod -aG kvm $USER
  groups
  ```

### JAVA_HOME Não Configurado
* **Instalação e Exportação:**
  ```bash
  sudo apt install openjdk-17-jdk
  echo "export JAVA_HOME=/usr/lib/jvm/java-17-openjdk-amd64" >> ~/.zshrc
  echo "export PATH=\$JAVA_HOME/bin:\$PATH" >> ~/.zshrc
  source ~/.zshrc
  ```

### Conflito Dual GPU NVIDIA + AMD no Wayland
* **Desabilitar iGPU AMD:**
  ```bash
  echo "blacklist amdgpu" | sudo tee /etc/modprobe.d/blacklist-amdgpu.conf
  echo "options nvidia-drm modeset=1" | sudo tee /etc/modprobe.d/nvidia-drm.conf
  sudo update-initramfs -u
  sudo reboot
  ```

### Timeout de GPU NVIDIA Durante Jogos (Tela Preta)
* **Ajuste no GRUB (`/etc/default/grub`):**
  ```text
  GRUB_CMDLINE_LINUX_DEFAULT="quiet splash nvidia-drm.modeset=1 nvidia.NVreg_PreserveVideoMemoryAllocations=1 nvidia.NVreg_EnableGpuFirmware=0"
  ```
  ```bash
  sudo update-grub && sudo reboot
  ```

### PATH Quebrado no Terminal
* **Recuperação de Emergência:**
  ```bash
  export PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:$PATH
  /usr/bin/sudo /usr/bin/apt install ubuntu-desktop gnome-session gnome-shell
  ```

### Limite de File Watchers Atingido (ENOSPC inotify)
* **Configuração Permanente em `/etc/sysctl.d/99-inotify.conf`:**
  ```ini
  fs.inotify.max_user_watches=524288
  fs.inotify.max_user_instances=1024
  fs.inotify.max_queued_events=32768
  ```
  ```bash
  sudo sysctl --system
  cat /proc/sys/fs/inotify/max_user_watches
  ```

### Ícone de Engrenagem/Catraca Duplicado no Ubuntu Dock (StartupWMClass)
* **Diagnóstico da Classe:**
  1. Pressione `Alt + F2`.
  2. Digite `lg` e pressione `Enter`.
  3. Na aba **Windows**, copie o valor de `wmclass` ou `app_id`.
  4. Pressione `Esc` para fechar.
* **Correção no Lançador (`~/.local/share/applications/`):**
  Adicione `StartupWMClass=classe-obtida` ao arquivo `.desktop`.
* **Atualização do Cache:**
  ```bash
  update-desktop-database ~/.local/share/applications/
  ```
