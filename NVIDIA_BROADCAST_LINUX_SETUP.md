# NVIDIA Broadcast para Linux + Integração Iriun Webcam

- **Repositório do Criador Original:** [Hkshoonya/nvidia-broadcast-linux](https://github.com/Hkshoonya/nvidia-broadcast-linux.git)
- **Caminho Local do Projeto:** `/home/data/projects/nvidia-broadcast-linux`

---

## 1. Problema de Conflito com Iriun Webcam & Como Resolvemos

### O Erro Observado:
Ao instalar e rodar o NVIDIA Broadcast pela primeira vez, o aplicativo **Iriun Webcam** parava de funcionar e exibia a tela de falha:

```
Initialization failed. v4l2loopback missing.
You may want to run:
sudo rmmod v4l2loopback; sudo modprobe v4l2loopback
```

### O Motivo Técnico:
1. **Conflito de Dispositivo de Vídeo:** O script de instalação padrão (`install.sh`) do NVIDIA Broadcast tentava regravar a configuração do módulo do kernel `v4l2loopback` para alocar apenas **1 dispositivo** na porta `/dev/video10`. Isso descarregava a porta `/dev/video0` que o Iriun Webcam usa para receber o sinal da câmera do celular.
2. **Filtro de Segurança Bloqueado:** No código-fonte do NVIDIA Broadcast (`src/nvbroadcast/video/virtual_camera.py`), a função `_is_virtual_camera_name` possuía uma trava de segurança com a string genérica `v4l2loopback`. Como o Iriun se registra como `Iriun Webcam (platform:v4l2loopback-000)`, o aplicativo ocultava o Iriun da lista de seleção de entrada (**Source**).
3. **Seletor de Câmeras Sem Padrão:** Na interface gráfica (`src/nvbroadcast/ui/window.py`), o campo **Source** exibia `(None)` quando o dispositivo salvo na sessão anterior não era encontrado, exigindo intervenção manual.

---

## 2. Modificações Realizadas no Código-Fonte

### A. Liberação da Câmera do Iriun
- **Arquivo:** `src/nvbroadcast/video/virtual_camera.py`
- **Modificação:** Removida a string `v4l2loopback` da tupla `_VIRTUAL_CAMERA_MARKERS`.
- **Resultado:** O NVIDIA Broadcast passa a listar o **Iriun Webcam** (`/dev/video0`) na seleção de fontes de entrada, mantendo apenas a sua própria saída (`NVbroadcast`) ocultada para não gerar loop.

### B. Seleção Automática na Interface
- **Arquivo:** `src/nvbroadcast/ui/window.py`
- **Modificação:** Adicionada a chamada `set_selected_index(0)` nas funções `_populate_devices` e `sync_video_input_controls`.
- **Resultado:** O aplicativo seleciona automaticamente a **Iriun Webcam** assim que a janela é iniciada.

---

## 3. Modo Temporário (Configuração em Memória RAM - Recomendado)

No modo temporário, **nenhum arquivo do sistema operacional é modificado permanentemente**. Ao reiniciar o computador, a memória RAM é limpa e o sistema retorna 100% ao estado original.

### Passo 1: Limpar arquivos de inicialização do disco
```bash
rm -f ~/.config/autostart/com.doczeus.NVBroadcast.desktop
sudo rm -f /etc/modprobe.d/nvbroadcast-v4l2loopback.conf /etc/modules-load.d/nvbroadcast-v4l2loopback.conf /etc/modprobe.d/v4l2loopback.conf /etc/modules-load.d/v4l2loopback.conf
```

### Passo 2: Carregar as 2 Câmeras Virtuais na RAM
Execute este comando para alocar a porta `/dev/video0` para o Iriun e a porta `/dev/video10` para o NVIDIA Broadcast simultaneamente:

```bash
sudo modprobe -r v4l2loopback
sudo modprobe v4l2loopback devices=2 video_nr=0,10 card_label="Iriun Webcam","NVbroadcast" exclusive_caps=1,1 max_buffers=4
```

### Passo 3: Executar a versão atualizada do aplicativo
```bash
cd /home/data/projects/nvidia-broadcast-linux
.venv/bin/python -m nvbroadcast
```

---

## 4. Modo Permanente (Configuração no Boot) & Como Remover

Se no futuro desejar que as duas portas de câmera virtuais sejam criadas automaticamente na inicialização do sistema:

### Criar os arquivos de configuração do kernel:
```bash
sudo bash -c 'echo "options v4l2loopback devices=2 video_nr=0,10 card_label=\"Iriun Webcam\",\"NVbroadcast\" exclusive_caps=1,1 max_buffers=4" > /etc/modprobe.d/v4l2loopback.conf'
sudo bash -c 'echo "v4l2loopback" > /etc/modules-load.d/v4l2loopback.conf'
```

### Como REMOVER a configuração permanente e voltar ao padrão:
Para apagar as configurações salvas em `/etc` e desfazer o modo permanente:
```bash
sudo rm -f /etc/modprobe.d/v4l2loopback.conf /etc/modules-load.d/v4l2loopback.conf
```

---

## 5. Fluxo de Uso em Aplicativos e Navegadores

```
[ Celular ] ────(Wi-Fi/USB)────▶ [ /dev/video0: Iriun Webcam ]
                                         │
                                         ▼
                                [ NVIDIA Broadcast ]
                              (Aplica IA / Blur / Denoise)
                                         │
                                         ▼
                                [ /dev/video10: NVbroadcast ]
                                         │
                                         ▼
                             [ Zoom / Meet / Discord / OBS ]
```

1. Ligue a câmera no aplicativo **Iriun Webcam** do celular e abra o Iriun no PC.
2. Abra o **NVIDIA Broadcast** (o campo **Source** exibirá automaticamente `Iriun Webcam (platform:v4l2loopback-000)`).
3. No seu navegador (Chrome/Edge/Brave) ou aplicativo de reunião (Zoom/Discord/Meet), selecione a câmera chamada **`NVbroadcast`** (`/dev/video10`).
4. **No Google Chrome:** Pressione **F5** na página da reunião para que o navegador atualize a lista de câmeras detectadas.
5. **No Firefox:** Ative a chave **`Firefox Mode`** na interface do NVIDIA Broadcast para ajustar o formato de cor para `I420`.

---

## 6. Galeria de Fundos Virtuais para Reuniões e Vídeos do YouTube

Todas as imagens de fundo foram organizadas no diretório:
`/home/data/projects/nvidia-broadcast-linux/.venv/share/nvbroadcast/backgrounds/`

### Cenários de Reunião e Escritório:
- `office_minimalist.jpg` — Escritório moderno minimalista
- `office_executive.jpg` — Sala de reunião executiva
- `office_bookshelf.jpg` — Estante de livros aconchegante
- `office_scandinavian.jpg` — Escritório estilo escandinavo
- `studio_dark.jpg` — Estúdio escuro profissional
- `office_loft.jpg` — Escritório amplo tipo loft
- `office_modern_desk.jpg` — Mesa de trabalho moderna
- `office_tech.jpg` — Ambiente corporativo tech
- `office_corporate.jpg` — Sala corporativa com luz natural
- `office_glass.jpg` — Escritório moderno com divisórias de vidro

### Cenários para Gravação de Vídeos no YouTube & Lives:
- `youtube_neon_studio.jpg` — Estúdio de gravação com iluminação Neon/RGB
- `youtube_streamer_room.jpg` — Quarto de streamer/criador de conteúdo
- `podcast_studio.jpg` — Estúdio de podcast profissional com isolamento acústico
- `developer_tech_setup.jpg` — Setup de desenvolvimento tech com múltiplos monitores
- `dark_library.jpg` — Biblioteca elegante com madeira escura
- `creative_bright_studio.jpg` — Estúdio criativo com iluminação natural
- `modern_architecture.jpg` — Interior arquitetônico moderno com grandes janelas
- `modern_villa_interior.jpg` — Interior de casa de alto padrão
- `video_recording_suite.jpg` — Suíte de gravação de vídeo profissional
- `scandinavian_living.jpg` — Sala de estar escandinava clean
- `cozy_living_room.jpg` — Sala de estar aconchegante com iluminação quente
- `minimalist_plants.jpg` — Cenário minimalista com plantas de ambiente
- `dark_aesthetic_studio.jpg` — Estúdio escuro com iluminação estética
- `lounge_coffee_space.jpg` — Espaço lounge estilo café moderno
- `startup_meeting_room.jpg` — Sala de reunião de startup
