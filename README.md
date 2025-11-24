# 🐧 Linux & Dev Handbook

Um guia de referência rápida para solução de problemas em Linux, desenvolvimento React Native, Docker e configurações de sistema. Este repositório serve como uma base de conhecimento (Knowledge Base) para desenvolvedores e sysadmins.

## 📋 Tabela de Conteúdos

- [Desenvolvimento Mobile](#-1-desenvolvimento-mobile-react-native--android)
- [Docker & Virtualização](#-2-docker--virtualização)
- [Git & Controle de Versão](#-3-git--controle-de-versão)
- [Linux System & Terminal](#-4-linux-system--terminal)
- [Hardware & Armazenamento](#-5-hardware--armazenamento)
- [Design & Web](#-6-design--web)

## 📱 1. Desenvolvimento Mobile (React Native & Android)

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
Para fazer o emulador funcionar corretamente com placas dedicadas (exemplo com AVD Experimental):
```bash
~/Android/sdk/emulator/emulator -avd Resizable_Experimental -gpu host
```

### Utilitários Android

**Gerar Keystore (Debug):**
```bash
keytool -genkey -v -keystore ./debug.keystore -keyalg RSA -keysize 2048 -validity 10000 -alias androiddebugkey -storepass android -keypass android
```

**Verificar SHA1 da Keystore:**
```bash
keytool -list -v -keystore debug.keystore -alias androiddebugkey -storepass android -keypass android
```

**Lançador Independente (Terminal):**
Use o `nohup` para rodar o emulador em background, liberando o terminal.
```bash
nohup ~/Android/sdk/emulator/emulator -avd Pixel_4_API_33 & disown
```

## 🐳 2. Docker & Virtualização

### Comandos Essenciais

| Ação | Comando |
|------|---------|
| Status (CPU/RAM) | `docker stats <container_id>` |
| Acessar Shell | `docker exec -it <container_id> /bin/bash` |
| Logs | `docker logs <container_id>` |
| Copiar (Host → Container) | `docker cp arquivo.txt <container_id>:/caminho/destino` |
| Copiar (Container → Host) | `docker cp <container_id>:/caminho/origem ./destino` |

### Docker OSX (MacOS em Container)

**Otimização de Performance:**
Edite o script de launch (`Launch.sh`):
```bash
RAM=16
CORES=8
SMP=8
```

**Aumentar memória com container rodando:**
```bash
docker update --memory=8g --memory-swap=8g <container_id>
```

### Armazenamento Docker (Mover para HD Externo)

1. Pare o Docker: `sudo systemctl stop docker`
2. Monte o HD e crie a pasta (Ex: `/media/hd-externo/docker`).
3. Transfira os dados: `rsync -aP /var/lib/docker/ /media/hd-externo/docker/`
4. Edite `/etc/docker/daemon.json`:
```json
{
   "data-root": "/media/hd-externo/docker"
}
```
5. Reinicie: `sudo systemctl start docker`

## 🐙 3. Git & Controle de Versão

### Configuração SSH (GitLab/GitHub)

**Gerar Chave Ed25519:**
```bash
ssh-keygen -t ed25519 -C "seu.email@dominio.com"
```

**Configurar `~/.ssh/config` (Evita conflitos):**
```bash
Host gitlab.com
   HostName gitlab.com
   User git
   IdentityFile ~/.ssh/id_ed25519
   IdentitiesOnly yes
```

### Limpeza e Manutenção

**Remover arquivos pesados do histórico (Permanente):**
⚠️ **Atenção:** Isso reescreve o histórico do Git.

```bash
# 1. Reescrever histórico removendo extensões específicas
git filter-branch --force --index-filter "git rm --cached --ignore-unmatch *.apk *.aab *.img *.jpg *.pdf *.mp4" --prune-empty --tag-name-filter cat -- --all

# 2. Limpar cache e referências antigas
git reflog expire --expire=now --all
git gc --prune=now --aggressive

# 3. Forçar envio
git push origin --force --all
```

## 🐧 4. Linux System & Terminal

### Gerenciamento de Shell (Python & Nohup)

**Instalação Python 3.10 (Ubuntu):**
```bash
sudo add-apt-repository ppa:deadsnakes/ppa
sudo apt update
sudo apt install python3.10 python3.10-venv python3.10-distutils
curl -sS https://bootstrap.pypa.io/get-pip.py | sudo python3.10
```

**Nohup (Processos em Background):**
O `nohup` mantém o processo rodando após logout.
- Rodar: `nohup comando > log.out 2>&1 &`
- Parar: `ps aux | grep comando` seguido de `kill <PID>`

### SSH & Acesso Remoto

- Instalar Server: `sudo apt install openssh-server`
- Montar Pasta (SSHFS): `sshfs user@servidor:/caminho/remoto ./pasta_local`
- Correção VS Code SSH: Adicione ao settings.json: `"remote.SSH.useLocalServer": false`

### Truques de Desktop (Gnome/Mint)

**Fixar Janela (Always on Top):**
```bash
wmctrl -r :SELECT: -b add,above
```

**Criar atalho "Novo Documento" no Nautilus:**
```bash
mkdir -p ~/Templates && touch ~/Templates/"Documento Vazio"
```

**Wake-on-LAN Permanente:**
Crie `/etc/systemd/system/wol-enable.service`:
```ini
[Unit]
Description=Enable Wake-on-LAN
After=network.target

[Service]
Type=oneshot
ExecStart=/sbin/ethtool -s enp6s0 wol g
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
```

Ative: `sudo systemctl enable wol-enable.service`

## 💾 5. Hardware & Armazenamento

### Montagem Automática de Discos (fstab)

1. Descubra o UUID: `sudo blkid`
2. Crie o ponto de montagem: `sudo mkdir -p /media/data/ntfs1`
3. Adicione ao `/etc/fstab`:
```bash
UUID=SEU_UUID_AQUI /media/data/ntfs1 ntfs-3g defaults 0 0
```
4. Teste: `sudo mount -a`

### Segurança (BitLocker & Hashcat)

**Windows:** Desativar BitLocker via Regedit
- Caminho: `HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\BitLocker`
- Chave: `PreventDeviceEncryption` (DWORD = 1)

**Hashcat:**
- Máscara: `hashcat -m 22000 arquivo.hc22000 -a 3 ?d?d?d?d?d?d?d?d`
- Complexo: `hashcat -m 22000 arquivo.hc22000 -a 3 '?l?u?d?l?u?d...' -o saida.txt`
- Logs: `sudo cat /root/.local/share/hashcat/hashcat.potfile`

## 🎨 6. Design & Web

- **Gerador de Gradiente CSS:** [Coolors Gradient Maker](https://coolors.co/gradient-maker)
