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
Originalmente, se o aplicativo iniciasse sem uma câmera previamente gravada na configuração, o campo **Source** ficava unselected em `(None)`.

**1ª Modificação (Linhas 2115 a 2119):**

```python
# CÓDIGO ORIGINAL (Antes):
def _populate_devices(self):
    cameras = list_camera_devices()
    if cameras:
        self._camera_selector.set_devices(cameras)

# CÓDIGO ATUALIZADO (Depois):
def _populate_devices(self):
    cameras = list_camera_devices()
    if cameras:
        self._camera_selector.set_devices(cameras)
        self._camera_selector.set_selected_index(0)
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

### C. Manutenção do Áudio Ativo em Tempo Real (`auto_idle = False`)
- **Arquivos:** `src/nvbroadcast/core/config.py` e `src/nvbroadcast/audio/pipeline.py`
- **Modificação:** O parâmetro padrão `auto_idle` (economia de energia do áudio) foi alterado de `True` para `False`.
- **Motivo:** Quando `auto_idle = True`, o pipeline de captura do microfone pausava automaticamente quando nenhum aplicativo externo (como Zoom ou Meet) estava gravando áudio da porta virtual. Isso fazia com que o medidor de decibéis na interface do NV Broadcast ficasse congelado em `-60 dB` (mudo). Com `auto_idle = False`, o microfone Fifine e o medidor de decibéis permanecem ativos e atualizando continuamente em tempo real na tela.

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
