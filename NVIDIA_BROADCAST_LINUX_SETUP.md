# NVIDIA Broadcast para Linux + Integração Iriun Webcam

- **Repositório do Criador Original:** [Hkshoonya/nvidia-broadcast-linux](https://github.com/Hkshoonya/nvidia-broadcast-linux.git)
- **Caminho Local do Projeto Atualizado:** `/home/data/projects/nvidia-broadcast-linux`

---

## 1. Entendendo as Alterações no Código-Fonte

As correções para o NVIDIA Broadcast funcionar com o Iriun Webcam foram feitas **diretamente no código-fonte Python do projeto clonado em `/home/data/projects/nvidia-broadcast-linux`**:

1. **`src/nvbroadcast/video/virtual_camera.py`**: Removida a trava genérica `v4l2loopback` da tupla `_VIRTUAL_CAMERA_MARKERS`. Isso permite que o aplicativo reconheça o **Iriun Webcam** (`/dev/video0`) na lista de entradas (**Source**), mantendo oculta apenas a sua própria saída de vídeo (`NVbroadcast`).
2. **`src/nvbroadcast/ui/window.py`**: Adicionada a instrução `set_selected_index(0)` para forçar a interface gráfica a selecionar o **Iriun Webcam** automaticamente ao abrir, sem deixar em `(None)`.

> **Importante:** Se você abrir uma versão oficial não modificada sem essas alterações de código, o Iriun Webcam não aparecerá na lista de entradas. Por isso, o ícone no menu do sistema (`~/.local/share/applications/nvbroadcast.desktop`) e o alias `nvbc` foram apontados para o código atualizado na pasta `/home/data/projects/nvidia-broadcast-linux`.

---

## 2. Configuração Permanente (Ativada no Sistema)

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

## 3. Como Abrir o Aplicativo

Como o atalho de menu e o alias já estão configurados, você **não precisa digitar o caminho longo no terminal**:

1. **Pelo Menu do Sistema:** Abra o menu de aplicativos do Ubuntu/Linux e clique no ícone **NVIDIA Broadcast**.
2. **Pelo Terminal:** Basta digitar `nvbc` e dar Enter.

---

## 4. Guia de Uso em Aplicativos e Navegadores

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

## 5. Galeria de Fundos Virtuais Adicionados (28 Imagens)

Todas as 28 imagens de fundo estão disponíveis em:
`/home/data/projects/nvidia-broadcast-linux/.venv/share/nvbroadcast/backgrounds/`

- **Reuniões e Escritórios:** `office_minimalist.jpg`, `office_executive.jpg`, `office_bookshelf.jpg`, `office_scandinavian.jpg`, `studio_dark.jpg`, `office_loft.jpg`, `office_modern_desk.jpg`, `office_tech.jpg`, `office_corporate.jpg`, `office_glass.jpg`.
- **Vídeos para o YouTube & Lives:** `youtube_neon_studio.jpg`, `youtube_streamer_room.jpg`, `podcast_studio.jpg`, `developer_tech_setup.jpg`, `dark_library.jpg`, `creative_bright_studio.jpg`, `modern_architecture.jpg`, `modern_villa_interior.jpg`, `video_recording_suite.jpg`, `scandinavian_living.jpg`, `cozy_living_room.jpg`, `minimalist_plants.jpg`, `dark_aesthetic_studio.jpg`, `lounge_coffee_space.jpg`, `startup_meeting_room.jpg`.
