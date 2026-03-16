# Linux & Dev Handbook

> Referência pessoal de setup, configurações e troubleshooting.

---

## Tabela de Conteúdos

### Setup & Configuração

- [1. Mobile](#1-mobile)
  - [Android SDK no Zsh](#android-sdk-no-zsh)
  - [Alias — Liberar RAM](#alias--liberar-ram)
- [2. Docker](#2-docker)
  - [Limpeza de Cache](#limpeza-de-cache)
  - [Docker no HD Externo](#docker-no-hd-externo)
- [3. Git](#3-git)
  - [SSH](#ssh)
  - [Remover Arquivos Pesados do Histórico](#remover-arquivos-pesados-do-histórico)
- [4. Linux](#4-linux)
  - [Oh My Zsh](#oh-my-zsh)
  - [SSH & Acesso Remoto](#ssh--acesso-remoto)
  - [Pendrive Bootável](#pendrive-bootável)
  - [Fixar Janela no Topo](#fixar-janela-no-topo)
  - [Atalhos de Teclado no Linux Mint](#atalhos-de-teclado-no-linux-mint)
- [5. Hardware & Armazenamento](#5-hardware--armazenamento)
  - [Montagem Automática de Discos](#montagem-automática-de-discos)
  - [Hashcat](#hashcat)
- [6. Bluetooth & Controles](#6-bluetooth--controles)
  - [DualShock 4 no Linux Mint](#dualshock-4-no-linux-mint-223)

### Troubleshooting

- [7. Mobile](#7-mobile-troubleshooting)
  - [Emulador travando com "Not Responding"](#emulador-travando-com-emulator-is-not-responding)
  - [JAVA_HOME não configurado](#java_home-não-configurado)
- [8. Linux](#8-linux-troubleshooting)
  - [PATH quebrado — sudo e apt não encontrados](#path-quebrado--sudo-e-apt-não-encontrados)

---

## Setup & Configuração

---

## 1. Mobile

### Android SDK no Zsh

Adicione ao `~/.zshrc`:

```bash
export ANDROID_HOME=$HOME/Android/Sdk
export PATH=$PATH:$ANDROID_HOME/tools:$ANDROID_HOME/platform-tools
```

### Alias — Liberar RAM

```bash
alias lp='sudo sync && echo 3 | sudo tee /proc/sys/vm/drop_caches'
```

---

## 2. Docker

### Limpeza de Cache

```bash
docker system prune -a --volumes -f && docker builder prune -a -f && docker compose build --no-cache && docker compose up -d
```

> Libera o espaço do cache de build inútil instantaneamente.

### Docker no HD Externo

O Docker é composto por dois serviços independentes que precisam ser configurados separadamente: o **dockerd** e o **containerd**. Se você configurar apenas um deles, o outro continuará gravando no SSD sem que você perceba.

**1. Verificar onde o HD está montado:**

```bash
df -h
```

**2. Parar os serviços:**

```bash
sudo systemctl stop docker docker.socket containerd
```

**3. Configurar o Docker — edite `/etc/docker/daemon.json`:**

```bash
sudo nano /etc/docker/daemon.json
```

```json
{
  "data-root": "/media/data/SEU-UUID-AQUI/docker"
}
```

**4. Configurar o containerd — gere e edite o `config.toml`:**

```bash
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml
sudo nano /etc/containerd/config.toml
```

Altere a linha `root` para:

```
root = "/media/data/SEU-UUID-AQUI/containerd"
```

> Deixe a variável `state` como está — ela aponta para `/run/containerd` (memória RAM) e não deve ser alterada.

**5. Mover os dados existentes para o HD (se necessário):**

```bash
sudo rsync -aP /var/lib/docker/ /media/data/SEU-UUID-AQUI/docker/
sudo rsync -aP /var/lib/containerd/ /media/data/SEU-UUID-AQUI/containerd/
```

**6. Iniciar os serviços:**

```bash
sudo systemctl start containerd && sudo systemctl start docker
```

**7. Confirmar:**

```bash
docker info | grep "Docker Root Dir"
```

**8. Apagar as pastas antigas do SSD (após confirmar que está funcionando):**

```bash
sudo rm -rf /var/lib/docker
sudo rm -rf /var/lib/containerd
```

---

## 3. Git

### SSH

```bash
ssh-keygen -t ed25519 -C "seu.email@dominio.com"
```

### Remover Arquivos Pesados do Histórico

```bash
git filter-branch --force --index-filter "git rm --cached --ignore-unmatch *.apk *.aab *.img *.jpg *.pdf *.mp4" --prune-empty --tag-name-filter cat -- --all
git reflog expire --expire=now --all
git gc --prune=now --aggressive
git push origin --force --all
```

---

## 4. Linux

### Oh My Zsh

Themes: https://github.com/ohmyzsh/ohmyzsh/wiki/Themes — tema recomendado: `bira`

**Reinstalar do zero:**

```bash
uninstall_oh_my_zsh && rm -f ~/.zshrc && sh -c "$(curl -fsSL https://raw.githubusercontent.com/ohmyzsh/ohmyzsh/master/tools/install.sh)"
```

**Autosuggestions:**

```bash
git clone https://github.com/zsh-users/zsh-autosuggestions ${ZSH_CUSTOM:-~/.oh-my-zsh/custom}/plugins/zsh-autosuggestions && sed -i 's/plugins=(git)/plugins=(git zsh-autosuggestions)/' ~/.zshrc && source ~/.zshrc
```

macOS:

```bash
git clone https://github.com/zsh-users/zsh-autosuggestions ${ZSH_CUSTOM:-~/.oh-my-zsh/custom}/plugins/zsh-autosuggestions && sed -i '' 's/plugins=(git)/plugins=(git zsh-autosuggestions)/' ~/.zshrc && source ~/.zshrc
```

**Mudar shell padrão:**

```bash
chsh -s $(which zsh)
sudo chsh -s $(which zsh) root
```

### SSH & Acesso Remoto

```bash
sudo apt install openssh-server
sshfs user@servidor:/caminho/remoto ./pasta_local
```

### Pendrive Bootável

**Balena Etcher** — recomendado: [etcher.balena.io](https://etcher.balena.io)

### Fixar Janela no Topo

```bash
sudo apt install wmctrl
wmctrl -r :SELECT: -b add,above     # fixar
wmctrl -r :SELECT: -b remove,above  # desfixar
```

### Atalhos de Teclado no Linux Mint

1. Menu → pesquise `Teclado`
2. Aba **Atalhos de Teclado**
3. Clique em **Adicionar atalho personalizado**
4. Cole o comando desejado e defina a tecla

> **Dica:** Combine com o `wmctrl` acima para fixar janelas com um atalho.

---

## 5. Hardware & Armazenamento

### Montagem Automática de Discos

```bash
sudo blkid  # descobrir UUID
sudo nano /etc/fstab
```

Adicione a linha:

```
UUID=SEU_UUID_AQUI /media/data/meu-hd ext4 defaults,nofail 0 2
```

### Hashcat

```bash
hashcat -m 22000 arquivo.hc22000 -a 3 ?d?d?d?d?d?d?d?d
```

---

## 6. Bluetooth & Controles

### DualShock 4 no Linux Mint 22.3

O controle conecta via Bluetooth mas o Steam não reconhece porque o BlueZ por padrão exige "bonding" completo para aceitar conexões HID. Controles genéricos e DS4 não fazem esse processo, ficando bloqueados mesmo aparecendo como conectados.

**Solução 1 — Controle não reconhecido pelo Steam**

Edite `/etc/bluetooth/input.conf`:

```bash
sudo nano /etc/bluetooth/input.conf
```

Mude para:

```
ClassicBondedOnly=false
```

> **Por quê?** Com `true` (padrão), o BlueZ só aceita dispositivos HID que completaram o processo de bonding. Controles genéricos e DS4 não fazem esse processo, ficando bloqueados mesmo aparecendo como conectados.

**Solução 2 — Controle não reconecta automaticamente**

Edite `/etc/bluetooth/main.conf`:

```bash
sudo nano /etc/bluetooth/main.conf
```

Na seção `[Policy]`, deixe assim:

```
AutoEnable=true
ReconnectAttempts=7
ReconnectIntervals=1,2,4,8,16,32,64
ReconnectUUIDs=00001112-0000-1000-8000-00805f9b34fb,0000111f-0000-1000-8000-00805f9b34fb,0000110a-0000-1000-8000-00805f9b34fb,0000110b-0000-1000-8000-00805f9b34fb
```

Depois de qualquer alteração, reinicie:

```bash
sudo systemctl restart bluetooth
```

Verifique se o controle foi reconhecido:

```bash
ls /dev/input/js*
# Deve retornar: /dev/input/js0
```

> No Linux Mint 23.x nenhuma dessas configurações é necessária — o padrão já funciona.

---

## Troubleshooting

---

## 7. Mobile Troubleshooting

### Emulador travando com "Emulator is Not Responding"

**Problema:** O emulador exibe a mensagem "Emulator is Not Responding" e trava constantemente, mesmo com hardware potente.

**Causa:** O `hw.gpu.mode=auto` faz o emulador hesitar entre software rendering e GPU real, ignorando a GPU dedicada e causando travamentos.

**Solução:** Abra o `config.ini` do AVD:

```bash
nano ~/.android/avd/NOME_DO_SEU_AVD.avd/config.ini
```

Localize e altere:

```
hw.gpu.mode=auto
```

Para:

```
hw.gpu.mode=host
```

Salve com **Ctrl+O** → Enter → **Ctrl+X**, feche e reabra o emulador.

# Adicionar ao grupo kvm
sudo usermod -aG kvm $USER

# Confirmar
groups
> Com `host`, o emulador usa diretamente a GPU real do sistema, eliminando os travamentos causados pela renderização por software.

---

### JAVA_HOME não configurado

**Problema:**

```
ERROR: JAVA_HOME is not set and no 'java' command could be found in your PATH.
```

**Solução:**

1. Instale o Java caso não tenha:

```bash
sudo apt install openjdk-17-jdk
```

2. Adicione ao `~/.zshrc` ou `~/.bashrc`:

```bash
export ANDROID_HOME=$HOME/Android/Sdk
export JAVA_HOME=/usr/lib/jvm/java-17-openjdk-amd64
export PATH=$JAVA_HOME/bin:$PATH
```

3. Aplique:

```bash
source ~/.zshrc
java -version
```

> Se o caminho não existir, verifique com `find /usr/lib/jvm -maxdepth 1 -type d` qual versão está instalada.

---

## 8. Linux Troubleshooting

### PATH quebrado — sudo e apt não encontrados

**Problema:** Após um `apt autoremove` acidental, comandos como `sudo`, `apt` e `su` param de funcionar com `command not found`.

**Causa:** O autoremove removeu dependências essenciais do sistema em cascata.

**Solução:**

1. Restaure o PATH da sessão atual:

```bash
export PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:$PATH
```

2. Reinstale os pacotes essenciais usando o caminho completo:

```bash
/usr/bin/sudo /usr/bin/apt install ubuntu-desktop gnome-session gnome-shell
```

3. Reinicie o sistema após a instalação.

> ⚠️ **Prevenção:** Nunca confirme um `apt autoremove` sem revisar a lista completa do que será removido. Se aparecer `gnome-session`, `gnome-shell` ou `xwayland` na lista, cancele com `n`.
