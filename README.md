# Linux & Dev Handbook

Um guia de referência rápida para solução de problemas em Linux, desenvolvimento React Native, Docker e configurações de sistema. Este repositório serve como uma base de conhecimento (Knowledge Base) para desenvolvedores e sysadmins.

## Tabela de Conteúdos

- [Desenvolvimento Mobile](#1-desenvolvimento-mobile-react-native--android)
- [Docker & Virtualização](#2-docker--virtualização)
- [Git & Controle de Versão](#3-git--controle-de-versão)
- [Linux System & Terminal](#4-linux-system--terminal)
- [Hardware & Armazenamento](#5-hardware--armazenamento)
- [Anydesk (Acesso Remoto)](#6-anydesk-acesso-remoto-linux-mint)
- [Bluetooth & Controles](#7-bluetooth--controles)
- [Design & Web](#8-design--web)

## 1. Desenvolvimento Mobile (React Native & Android)

### Configuração de Ambiente (Android SDK)

Adicione estas linhas ao seu arquivo de configuração de shell (`~/.bashrc`, `~/.zshrc` ou `~/.config/fish/config.fish`).

**Bash / Zsh:**
```bash
export ANDROID_HOME=$HOME/Android/sdk
export PATH=$PATH:$ANDROID_HOME/tools:$ANDROID_HOME/platform-tools
```

**Fish Shell:**
```bash
set -Ux ANDROID_HOME $HOME/Android/sdk
set -Ux PATH $PATH $ANDROID_HOME/tools $ANDROID_HOME/platform-tools
```

### Builds com Expo (EAS)

**Build Local:**
```bash
# Android (com medição de tempo)
time eas build --local --platform android --profile preview

# iOS
eas build --local --platform ios --profile preview
```

**Build via Servidor Expo:**
```bash
eas build -p android --profile preview
```

### Solução de Erros Comuns

**Erro Gradle / Java Version:** Se ocorrer erro de `compileReleaseJavaWithJavac`, atualize o Java (Geralmente do JDK 11 para 17).
```bash
sudo apt update && sudo apt install openjdk-17-jdk
```

**Emulador travando (Check Alive Timeout):** Remove o erro de "App is not responding" no GNOME.
```bash
gsettings set org.gnome.mutter check-alive-timeout 0
```

**Forçar GPU Dedicada (Ex: Nvidia):**
```bash
~/Android/sdk/emulator/emulator -avd Resizable_Experimental -gpu host
```

### Utilitários Android

**Gerar Keystore (Debug):**
```bash
keytool -genkey -v -keystore ./debug.keystore -keyalg RSA -keysize 2048 -validity 10000 -alias androiddebugkey -storepass android -keypass android
```

**Lançador Independente (Terminal):**
```bash
nohup ~/Android/sdk/emulator/emulator -avd Pixel_4_API_33 & disown
```

## 2. Docker & Virtualização

### Comandos Essenciais

## Docker – Comandos Úteis (Linux)

| Ação | Comando |
|------|---------|
| Status (CPU/RAM) | `docker stats <container_id>` |
| Acessar Shell | `docker exec -it <container_id> /bin/bash` |
| Logs | `docker logs <container_id>` |
| Logs (tempo real) | `docker logs -f <container_id>` |
| Copiar (Host → Container) | `docker cp arquivo.txt <container_id>:/caminho/destino` |
| Copiar (Container → Host) | `docker cp <container_id>:/caminho/arquivo.txt ./` |
| Iniciar Container | `docker start <container_id>` |
| Parar Container | `docker stop <container_id>` |
| Reiniciar Container | `docker restart <container_id>` |
| Remover Container | `docker rm <container_id>` |
| Remover Container (forçado) | `docker rm -f <container_id>` |
| Listar Containers ativos | `docker ps` |
| Listar Todos os Containers | `docker ps -a` |
| Listar Imagens | `docker images` |
| Remover Imagem | `docker rmi <image_id>` |
| Ver Uso de Disco | `docker system df` |
| Limpar Cache de Build | `docker builder prune -a -f` |
| Limpeza Geral (⚠️ tudo não usado) | `docker system prune -a -f` |
| Inspecionar Container | `docker inspect <container_id>` |
| Ver Variáveis de Ambiente | `docker exec <container_id> env` |
| Ver Portas Expostas | `docker port <container_id>` |
| Atualizar policy de restart | `docker update --restart=no <container_id>` |
| Atualizar policy de restart | `docker update --restart=on-failure <container_id>` |
| Atualizar policy de restart | `docker update --restart=unless-stopped <container_id>` |
| Ver restart policy (todos) | `docker inspect $(docker ps -aq) --format '{{.Name}} → {{.HostConfig.RestartPolicy.Name}}'` |



> **Dica:** O comando `docker builder prune -a -f` libera o espaço do cache inútil instantaneamente.

### Docker OSX (MacOS em Container)

**Otimização de Performance:**
Edite o script de launch (`Launch.sh`):
```bash
RAM=16
CORES=8
SMP=8
```

### Armazenamento Docker (Mover para HD Externo)

1. Pare o Docker: `sudo systemctl stop docker`
2. Transfira os dados: `rsync -aP /var/lib/docker/ /media/hd-externo/docker/`
3. Edite `/etc/docker/daemon.json`:
```json
{
    "data-root": "/media/hd-externo/docker"
}
```
4. Reinicie: `sudo systemctl start docker`

## 3. Git & Controle de Versão

### Configuração SSH (GitLab/GitHub)

**Gerar Chave Ed25519:**
```bash
ssh-keygen -t ed25519 -C "seu.email@dominio.com"
```

### Limpeza e Manutenção

**Remover arquivos pesados do histórico (Permanente):**
```bash
git filter-branch --force --index-filter "git rm --cached --ignore-unmatch *.apk *.aab *.img *.jpg *.pdf *.mp4" --prune-empty --tag-name-filter cat -- --all
git reflog expire --expire=now --all
git gc --prune=now --aggressive
git push origin --force --all
```

## 4. Linux System & Terminal

### Oh My Zsh Setup
themes https://github.com/ohmyzsh/ohmyzsh/wiki/Themes
perfect theme bira.

## Reinstalar

**Reinstalar do Zero:**
```bash
uninstall_oh_my_zsh && rm -f ~/.zshrc && sh -c "$(curl -fsSL https://raw.githubusercontent.com/ohmyzsh/ohmyzsh/master/tools/install.sh)"
```

**Adicionar Autosuggestions (Linux):**
```bash
git clone https://github.com/zsh-users/zsh-autosuggestions ${ZSH_CUSTOM:-~/.oh-my-zsh/custom}/plugins/zsh-autosuggestions && sed -i 's/plugins=(git)/plugins=(git zsh-autosuggestions)/' ~/.zshrc && source ~/.zshrc
```

**Adicionar Autosuggestions (macOS com Homebrew):**
```bash
brew install zsh-autosuggestions
echo 'source $(brew --prefix)/share/zsh-autosuggestions/zsh-autosuggestions.zsh' >> ~/.zshrc
source ~/.zshrc
```


Para seu usuário:
```bash
chsh -s $(which zsh)
```
Para o root:
```bash
sudo chsh -s $(which zsh) root
```

### Gerenciamento de Shell (Python & Nohup)

**Instalação Python 3.10 (Ubuntu):**
```bash
sudo add-apt-repository ppa:deadsnakes/ppa
sudo apt update
sudo apt install python3.10 python3.10-venv python3.10-distutils
curl -sS https://bootstrap.pypa.io/get-pip.py | sudo python3.10
```

### SSH & Acesso Remoto

- Instalar Server: `sudo apt install openssh-server`
- Montar Pasta (SSHFS): `sshfs user@servidor:/caminho/remoto ./pasta_local`

### Pendrives Bootáveis

- **Balena Etcher:** Ferramenta recomendada para criar pendrives bootáveis de forma segura. [Download aqui](https://etcher.balena.io).

### Fixar Janela no Topo (Always on Top)

Para deixar qualquer janela sempre visível sobre as outras, use o `wmctrl`:

**Instalar:**
```bash
sudo apt install wmctrl
```

**Fixar a janela selecionada:**
```bash
wmctrl -r :SELECT: -b add,above
```

> Ao rodar o comando, clique na janela que deseja fixar.

**Desfixar:**
```bash
wmctrl -r :SELECT: -b remove,above
```

### Criar Atalhos de Teclado no Linux Mint

1. Abra o **Menu** e pesquise por `Teclado`
2. Vá na aba **Atalhos de Teclado**
3. Clique em **Adicionar atalho personalizado**
4. No campo comando, cole o comando desejado (ex: o `wmctrl` acima)
5. Clique no campo de tecla e pressione a combinação desejada

> **Dica:** Combinando os dois — crie um atalho de teclado para o comando `wmctrl -r :SELECT: -b add,above` e fixe qualquer janela com um atalho personalizado.

## 5. Hardware & Armazenamento

### Montagem Automática de Discos (fstab)

1. Descubra o UUID: `sudo blkid`
2. Adicione ao `/etc/fstab`:
```bash
UUID=SEU_UUID_AQUI /media/data/ntfs1 ntfs-3g defaults 0 0
```

### Segurança (BitLocker & Hashcat)

**Hashcat (Ataque de Máscara):**
```bash
hashcat -m 22000 arquivo.hc22000 -a 3 ?d?d?d?d?d?d?d?d
```

## 6. Anydesk (Acesso Remoto Linux Mint)

### Configuração de Acesso Não Supervisionado

```bash
# Permitir acesso ao servidor X
xhost +

# Abrir configurações administrativas (Correção Qt)
sudo QT_QPA_PLATFORM=xcb anydesk --admin-settings
```

## 7. Bluetooth & Controles

### DualShock 4 (original ou genérico) via Bluetooth no Linux Mint 22.3

> **Contexto:** No Linux Mint 22.3 o controle conecta via Bluetooth normalmente, mas o Steam e o sistema não reconhecem como joystick. Isso acontece porque o BlueZ por padrão exige "bonding" completo para aceitar conexões HID, e controles genéricos (e alguns originais) não fazem esse processo.

#### 1. Carregar os módulos do kernel

```bash
sudo modprobe hid-sony
sudo modprobe hid-playstation
```

#### 2. Adicionar regras de udev

```bash
sudo nano /etc/udev/rules.d/50-ds4drv.rules
```

Cole o conteúdo abaixo:

```
KERNEL=="uinput", MODE="0666"
KERNEL=="hidraw*", SUBSYSTEM=="hidraw", ATTRS{idVendor}=="054c", ATTRS{idProduct}=="05c4", MODE="0666"
KERNEL=="hidraw*", SUBSYSTEM=="hidraw", ATTRS{idVendor}=="054c", ATTRS{idProduct}=="09cc", MODE="0666"
```

Recarregue as regras:

```bash
sudo udevadm control --reload-rules && sudo udevadm trigger
```

#### 3. Desativar restrição de bonding no BlueZ

Edite o arquivo de configuração do input:

```bash
sudo nano /etc/bluetooth/input.conf
```

Localize a linha:
```
#ClassicBondedOnly=true
```

Mude para:
```
ClassicBondedOnly=false
```

> **Por quê?** Com `true` (padrão), o BlueZ só aceita dispositivos HID que completaram o processo de bonding com troca de chaves de segurança. Controles genéricos e DS4 não fazem esse processo corretamente, ficando bloqueados mesmo conectados.

#### 4. Reiniciar o serviço Bluetooth

```bash
sudo systemctl restart bluetooth
```

#### 5. Confiar no dispositivo e conectar

```bash
bluetoothctl trust SEU:MAC:AQUI
bluetoothctl connect SEU:MAC:AQUI
```

> Para descobrir o MAC: `bluetoothctl devices`

#### 6. Verificar se funcionou

```bash
ls /dev/input/js*
# Deve retornar: /dev/input/js0
```

Se aparecer `js0`, o controle foi reconhecido com sucesso. Abra o Steam e ele deve aparecer automaticamente.

#### Observações

- No **Linux Mint 23.x** isso não é necessário pois o padrão do BlueZ já vem configurado de forma mais permissiva.
- Controles genéricos que se identificam como Sony DS4 (`usb:v054Cp09CC`) são suportados pelo módulo `hid_playstation` e podem mudar de cor ao conectar — isso é normal e indica que o módulo assumiu o controle corretamente.
- Para verificar o ID do dispositivo: `bluetoothctl info SEU:MAC:AQUI | grep Modalias`

## 8. Design & Web

- **Next.js:** Framework React focado em performance e SEO.
```bash
npx create-next-app@latest
```
