# NVIDIA Broadcast para Linux + Integração Iriun Webcam

- **Repositório do Criador Original:** [Hkshoonya/nvidia-broadcast-linux](https://github.com/Hkshoonya/nvidia-broadcast-linux.git)
- **Caminho Local do Projeto Atualizado:** `/home/data/projects/nvidia-broadcast-linux`

---

## 1. Explicação Detalhada das Alterações no Código-Fonte

As modificações para o NVIDIA Broadcast funcionar com o Iriun Webcam foram realizadas diretamente no código-fonte Python localizado em `/home/data/projects/nvidia-broadcast-linux`:

### A. Liberação da Câmera no Arquivo `src/nvbroadcast/video/virtual_camera.py`
O código original usava a trava genérica `v4l2loopback` para esconder câmeras virtuais de saída. Como o Iriun se identifica no sistema como `Iriun Webcam (platform:v4l2loopback-000)`, a função de busca ocultava a entrada do Iriun.

**Trecho modificado (Linhas 27 a 33):**

```python
# CÓDIGO ORIGINAL (Antes):
_VIRTUAL_CAMERA_MARKERS = (
    "v4l2loopback",
    "nvidia broadcast",
    "nvbroadcast",
    "obs virtual camera",
)

# CÓDIGO ATUALIZADO (Depois):
_VIRTUAL_CAMERA_MARKERS = (
    "nvidia broadcast",
    "nvbroadcast",
    "obs virtual camera",
)
```
* **Efeito:** Ao remover `"v4l2loopback"` da tupla, a função `list_camera_devices()` passou a reconhecer o Iriun Webcam (`/dev/video0`) na lista de seleção (**Source**), mantendo bloqueada apenas a saída própria do app (`NVbroadcast`).

---

### B. Seleção Automática na Interface no Arquivo `src/nvbroadcast/ui/window.py`
Originalmente, se o aplicativo iniciasse sem uma câmera ou microfone previamente salvos, o seletor de microfone marcava a opção mutada `Loopback Analog Stereo` por padrão em vez do microfone físico.

**Correção da Câmera (Linhas 2115 a 2119):**

```python
def _populate_devices(self):
    cameras = list_camera_devices()
    if cameras:
        self._camera_selector.set_devices(cameras)
        self._camera_selector.set_selected_index(0)
```

**Correção do Microfone (Linhas 1665 a 1680):**

```python
def _populate_mics(self):
    mics = self._app.list_microphones()
    self._mic_selector.set_devices(mics)
    # Ignora o Loopback virtual e seleciona o microfone físico Fifine
    for i, m in enumerate(mics):
        if "loopback" not in m.get("device", "").lower():
            self._mic_selector.set_selected_index(i)
            self._app.set_microphone(m.get("device", ""))
            break
```

**2ª Modificação (Linhas 2269 a 2279):**

```python
# CÓDIGO ORIGINAL (Antes):
camera_device = config.video.camera_device
for i, d in enumerate(getattr(self._camera_selector, "_devices", [])):
    if d["device"] == camera_device:
        self._camera_selector.set_selected_index(i)
        break

# CÓDIGO ATUALIZADO (Depois):
camera_device = config.video.camera_device
devices = getattr(self._camera_selector, "_devices", [])
selected_idx = 0
for i, d in enumerate(devices):
    if d["device"] == camera_device:
        selected_idx = i
        break
if devices:
    self._camera_selector.set_selected_index(selected_idx)
```
* **Efeito:** A interface gráfica agora força o seletor a marcar automaticamente a primeira câmera disponível (`index 0`), que é a **Iriun Webcam**, assim que a janela é aberta.

---

### D. Ativação do VU Meter em Tempo Real (`use_helper_process = False`)
- **Arquivo:** `src/nvbroadcast/audio/pipeline.py`
- **Modificação:** Alterada a variável `use_helper_process` de `True` para `False`.
- **Motivo:** No Linux, o aplicativo tentava isolar o áudio em um subprocesso secundário (`service.py`). No entanto, esse subprocesso isolado não enviava os dados de áudio de volta para a interface gráfica, impedindo que a barra de VU meter e o medidor de decibéis fossem atualizados na tela. Ao definir `use_helper_process = False`, o áudio roda de forma direta e os decibéis passam a responder instantaneamente conforme a sua voz.

---

### E. Correção Automática do Perfil do Microfone Fifine (`devices.py`)
- **Arquivo:** `src/nvbroadcast/audio/devices.py`
- **Diagnóstico:** O Linux criava dois perfis para o microfone Fifine:
  1. `fifine Microphone Digital Stereo (IEC958)` — Porta S/PDIF digital fictícia (sem sinal físico de som, resultando em `-60 dB`).
  2. `fifine Microphone Analog Stereo` — Porta analógica física real por onde passa o som da voz.
- **Modificação:** Adicionada uma verificação em `list_microphones()` que força o PipeWire (`pactl set-card-profile`) a ativar a porta analógica física real e remove a porta digital fictícia `iec958` da seleção.

---

## 2. Solução do Problema do Microfone Fifine (Áudio em -60 dB)

### O Problema:
O indicador de áudio do microfone Fifine ficava mudo em **-60 dB** na interface do NVIDIA Broadcast.

### O Motivo Técnico:
No gerenciador de áudio do Linux (PipeWire / PulseAudio), o volume do dispositivo de captura `alsa_input.usb-3142_fifine_Microphone-00.analog-stereo` estava reduzido para apenas **20% (-41.43 dB)**. Como o volume de captura estava extremamente baixo, a funcionalidade de **AI Denoiser (DeepFilterNet)** do NVIDIA Broadcast identificou o som fraco como ruído de fundo e o suprimiu totalmente, resultando em silêncio (-60 dB).

### A Solução:
Aumentamos o volume de captura do microfone Fifine no PipeWire para 100% (0.00 dB):

```bash
pactl set-source-volume alsa_input.usb-3142_fifine_Microphone-00.analog-stereo 100%
```

---

## 3. Configuração Permanente (Ativada no Sistema)

Com a **solução permanente**, você pode reiniciar o computador quantas vezes quiser. As duas portas de câmera virtuais serão criadas automaticamente pelo sistema durante o boot do Linux.

### Passo 1: Gravar a configuração permanente no kernel
Execute o comando abaixo uma única vez no terminal:

```bash
sudo bash -c 'echo "options v4l2loopback devices=2 video_nr=0,10 card_label=\"Iriun Webcam\",\"NVbroadcast\" exclusive_caps=1,1 max_buffers=4" > /etc/modprobe.d/v4l2loopback.conf' && sudo bash -c 'echo "v4l2loopback" > /etc/modules-load.d/v4l2loopback.conf'
```

### O que essa configuração permanente faz no boot:
- Aloca a porta `/dev/video0` para o **Iriun Webcam** (sua câmera de entrada vinda do celular).
- Aloca a porta `/dev/video10` para o **NVbroadcast** (sua câmera final tratada pela IA).

---

## 4. Como Abrir o Aplicativo

Como o atalho de menu e o alias já estão configurados, você **não precisa digitar o caminho longo no terminal**:

1. **Pelo Menu do Sistema:** Abra o menu de aplicativos do Ubuntu/Linux e clique no ícone **NVIDIA Broadcast**.
2. **Pelo Terminal:** Basta digitar `nvbc` e dar Enter.

---

## 5. Guia de Uso em Aplicativos e Navegadores

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

1. Abra o **Iriun Webcam** no celular e no PC.
2. Clique no ícone do **NVIDIA Broadcast** (o campo **Source** já estará com `Iriun Webcam`).
3. No **Zoom / Meet / Discord / OBS**, selecione a câmera **`NVbroadcast`**.
4. **No Google Chrome:** Pressione **F5** na página para atualizar a lista de câmeras.
5. **No Firefox:** Ative a chave **`Firefox Mode`** na interface do NV Broadcast.

---

## 6. Galeria de Fundos Virtuais Adicionados (28 Imagens)

Todas as 28 imagens de fundo estão disponíveis em:
`/home/data/projects/nvidia-broadcast-linux/.venv/share/nvbroadcast/backgrounds/`

- **Reuniões e Escritórios:** `office_minimalist.jpg`, `office_executive.jpg`, `office_bookshelf.jpg`, `office_scandinavian.jpg`, `studio_dark.jpg`, `office_loft.jpg`, `office_modern_desk.jpg`, `office_tech.jpg`, `office_corporate.jpg`, `office_glass.jpg`.
- **Vídeos para o YouTube & Lives:** `youtube_neon_studio.jpg`, `youtube_streamer_room.jpg`, `podcast_studio.jpg`, `developer_tech_setup.jpg`, `dark_library.jpg`, `creative_bright_studio.jpg`, `modern_architecture.jpg`, `modern_villa_interior.jpg`, `video_recording_suite.jpg`, `scandinavian_living.jpg`, `cozy_living_room.jpg`, `minimalist_plants.jpg`, `dark_aesthetic_studio.jpg`, `lounge_coffee_space.jpg`, `startup_meeting_room.jpg`.
- **Molduras e Overlays Transparentes (PNG):** `transparent_cyberpunk_neon_frame.png`, `transparent_studio_light_overlay.png`, `transparent_gamer_stream_frame.png`, `transparent_minimalist_glass_border.png`, `transparent_podcast_neon_sign.png`.

---

## 7. Alternativas Open-Source ao Iriun Webcam (Android e iPhone / iOS)

### A. scrcpy (100% Open-Source para Android via USB em 1080p/4K @ 60 FPS)
- **Vantagem:** Não precisa instalar nenhum aplicativo no celular. Usa a câmera nativa do Android via protocolo ADB USB com baixíssima latência e sem limites de qualidade.
- **Comando no Linux:**
  ```bash
  scrcpy --video-source=camera --camera-size=1920x1080 --camera-fps=60 --camera-facing=back --v4l2-sink=/dev/video0 --no-playback
  ```

### B. VDO.Ninja (100% Open-Source e Gratuito para iPhone / iOS e Android)
- **Vantagem:** Funciona em iPhones sem instalar nenhum aplicativo da App Store (roda nativamente via WebRTC no navegador Safari do iOS).
- **Latência:** Entre 30 ms e 80 ms (imperceptível) em Full HD 1080p / 60 FPS.
- **Passo a Passo de Uso:**
  1. **No iPhone/Celular:** Abra o **Safari**, acesse `https://vdo.ninja`, toque em **`Add your Camera to OBS`**, permita o acesso à câmera e toque em **`Start`**.
  2. **No PC Linux:** Acesse o link de visualização gerado (`https://vdo.ninja/?view=NOME_UNICO`).

---

## 8. Integração Avançada: VDO.Ninja + OBS Studio + NVIDIA Broadcast (Técnica das 2 Cenas)

Para usar o VDO.Ninja junto com os efeitos de IA do NVIDIA Broadcast no OBS sem gerar loops de vídeo ou travamento de câmera, utiliza-se a técnica de **Cena Oculta de Saída**:

### Arquitetura do Fluxo de Vídeo:
```
[ Celular / VDO.Ninja ]
         │
         ▼
[ OBS: Cena "Câmera Original" ] ──(Câmera Virtual: Scene Output)──▶ [ /dev/video0 ]
                                                                            │
                                                                            ▼
                                                                   [ NVIDIA Broadcast ]
                                                                 (Efeitos IA / Fundo)
                                                                            │
                                                                            ▼
                                                                   [ /dev/video10 ]
                                                                            │
                                                                            ▼
[ OBS: Cena Principal ] ◀──────(Dispositivo de Captura de Vídeo)───────────┘
```

### Configuração Passo a Passo no OBS Studio:

1. **Criar a Cena da Câmera Limpa (`Câmera Original`):**
   - No OBS, crie uma nova cena chamada `Câmera Original`.
   - Adicione uma fonte do tipo **Browser** com a URL do VDO.Ninja e ajuste para `1920x1080`.
   - Dê dois cliques na fonte `Browser`, desça até o final e **desmarque**:
     - `Shutdown source when not visible` (Desativar fonte quando invisível)
     - `Refresh browser when scene becomes active` (Atualizar o navegador quando a cena se tornar ativa)

2. **Configurar a Câmera Virtual do OBS para Transmitir a Cena Oculta:**
   - No painel de **Controls** (canto inferior direito), clique na **engrenagem ⚙️** ao lado de *Start Virtual Camera*.
   - Mude **Output Type** para `Scene`.
   - Em **Output Selection**, selecione `Câmera Original`.
   - Clique em **OK** e depois em **Start Virtual Camera**.

3. **Montar a Cena Principal de Transmissão:**
   - Volte para a sua cena principal (`Scene`).
   - Adicione uma nova fonte do tipo **Dispositivo de Captura de Vídeo (V4L2)** selecionando `/dev/video10` (`NVbroadcast`).
   - Agora você pode posicionar sua câmera com fundo recortado/blur na tela do OBS sem quebrar a transmissão nem gerar loops de vídeo!

---

## 9. Arquitetura Nativa Leve: Google Meet + OBS Studio + 2ª Câmera Virtual (`/dev/video1`)

Esta arquitetura dispensa o uso de softwares pesados proprietários e utiliza o próprio motor WebGL do **Google Meet** no navegador para trocar o fundo da câmera em 60 FPS, transmitindo a imagem tratada para a gravação no OBS Studio e para outros programas (Zoom, WhatsApp Web, Discord) através de uma 2ª Câmera Virtual.

```
[ Celular / VDO.Ninja ] ──▶ [ OBS: Cena "cameras" ] ──(Câmera Virtual 1: /dev/video0)──▶ [ Google Meet ]
                                                                                               │
                                                                                    (Aplica Fundo Virtual)
                                                                                               │
                                                                                               ▼
[ Zoom / WhatsApp / Apps ] ◄──(Câmera Virtual 2: /dev/video1) ◄──[ OBS: Window Capture ] ◄──────┘
```

### A. Configuração das 2 Câmeras Virtuais Permanentes no Linux
Para garantir que o Linux inicie sempre com as duas portas ativas no boot (`/dev/video0` para o celular limpo e `/dev/video1` para a imagem tratada do Meet):

1. **Editar o arquivo de opções do driver (`/etc/modprobe.d/iriunwebcam-options.conf`):**
   ```bash
   sudo bash -c 'echo "options v4l2loopback exclusive_caps=1,1 devices=2 video_nr=0,1 card_label=\"Iriun Webcam\",\"Segunda Camera\"" > /etc/modprobe.d/iriunwebcam-options.conf'
   ```

2. **Recarregar o driver na sessão atual:**
   ```bash
   sudo modprobe -r v4l2loopback 2>/dev/null || true && sudo modprobe v4l2loopback
   ```

### B. Passo a Passo no OBS Studio:

1. **Cena 1 (`cameras`):**
   - Adicione a fonte **Browser** com a URL do VDO.Ninja (`1920x1080`).
   - Na engrenagem **⚙️** ao lado de *Start Virtual Camera*, defina **Output Type**: `Scene` e **Output Selection**: `cameras`.
   - Clique em **Start Virtual Camera** (envia o sinal para a `/dev/video0` - `Iriun Webcam`).

2. **Google Meet:**
   - Entre no Meet e escolha a câmera `Iriun Webcam` (`/dev/video0`).
   - Escolha o fundo virtual/desfoque nativo do Meet.

3. **Cena 2 (`Scene` - Gravação e Transmissão):**
   - Adicione a fonte **Window Capture (PipeWire)** selecionando a janela do Google Meet.
   - Segure **Alt** e arraste as bordas para recortar apenas o retângulo do seu vídeo.
   - Para transmitir essa imagem do Meet para outros aplicativos (Zoom, WhatsApp Web, Discord):
     1. Clique com botão direito na fonte **Window Capture (PipeWire)** ➔ **Filters** (Filtros).
     2. Adicione o filtro **V4L2 Dedicated Output**.
     3. Selecione o dispositivo `/dev/video1` (`Segunda Camera`) e clique em **Start**.

