# ASUS V16 V3607VU com Arch Linux e Niri

## Objetivo

Registrar a configuração e os testes básicos do Arch Linux com Niri no ASUS V16, além dos pontos que precisam ser revistos ao migrar de um Acer Nitro 5.

O nome informado pelo firmware desta unidade é `ASUS Vivobook 16 V3607VU_V3607VU`. Os comandos devem ser conferidos antes de serem reutilizados em outra variante do V16, pois CPU, GPU, painel e placa de rede podem mudar entre modelos.

## Ambiente verificado

Verificado em 17 de outubro de 2026:

| Componente | Dispositivo |
| --- | --- |
| Modelo | ASUS V3607VU |
| Firmware | V3607VU.304 |
| CPU | Intel Core 5 210H, 8 núcleos e 12 threads |
| GPU integrada | Intel Raptor Lake-P Graphics, driver `i915` |
| GPU dedicada | NVIDIA GeForce RTX 4050 Laptop, driver `nvidia` |
| Tela interna | 1920×1200 a 144 Hz, com VRR |
| Wi-Fi | Realtek RTL8852BE, driver `rtw89_8852be` |
| Áudio | Intel cAVS com SOF |
| SSD | ADATA IM2P33F8 de 512 GB |
| Memória | 16 GiB |
| Sessão | Wayland com Niri 26.04 |
| Kernel | Linux 7.2.9.arch1-1 |

Obtenha os identificadores da sua unidade sem depender do nome comercial:

```bash
cat /sys/class/dmi/id/sys_vendor
cat /sys/class/dmi/id/product_name
cat /sys/class/dmi/id/bios_version
lspci -nnk
```

## Pacotes usados

Esta unidade usa o módulo aberto da NVIDIA com DKMS e o microcódigo da Intel:

```bash
sudo pacman -S \
  intel-ucode linux-firmware sof-firmware alsa-ucm-conf \
  mesa vulkan-intel \
  nvidia-open-dkms nvidia-utils lib32-nvidia-utils nvidia-prime \
  niri xorg-xwayland brightnessctl
```

O pacote `nvidia-open-dkms` exige os cabeçalhos de cada kernel instalado. Para o kernel padrão:

```bash
sudo pacman -S linux-headers
```

Se outro kernel for usado, instale os cabeçalhos correspondentes. Por exemplo, `linux-lts-headers` para `linux-lts`.

Após instalar ou atualizar o driver, reconstrua a imagem de inicialização e reinicie:

```bash
sudo mkinitcpio -P
sudo reboot
```

## NVIDIA no Wayland

O DRM KMS está habilitado nesta instalação por dois caminhos:

- parâmetro `nvidia_drm.modeset=1` na linha de comando do kernel;
- `options nvidia-drm modeset=1` em `/etc/modprobe.d/nvidia.conf`.

Nas versões atuais do driver, o parâmetro normalmente já é ativado por padrão. O importante é confirmar o estado, sem duplicar opções apenas por precaução:

```bash
cat /sys/module/nvidia_drm/parameters/modeset
cat /sys/module/nvidia_drm/parameters/fbdev
```

Os dois valores retornaram `Y` no ambiente verificado.

Confirme também quais drivers estão ligados às duas GPUs:

```bash
lspci -nnk | grep -A3 -E 'VGA|3D'
nvidia-smi
```

### Escolher entre NVIDIA global e modo híbrido

Esta instalação está configurada para usar a NVIDIA como renderizador OpenGL padrão. Isso foi confirmado por:

```bash
glxinfo -B | grep -E 'OpenGL vendor|OpenGL renderer'
```

As variáveis abaixo forçam esse comportamento quando definidas globalmente em `/etc/environment` ou no bloco `environment` do Niri:

```text
__NV_PRIME_RENDER_OFFLOAD=1
__GLX_VENDOR_LIBRARY_NAME=nvidia
GBM_BACKEND=nvidia-drm
```

Usar a RTX 4050 por padrão favorece jogos, monitor externo e compatibilidade, mas aumenta o consumo de bateria. Para priorizar autonomia, não force essas variáveis globalmente: deixe a sessão na GPU Intel e execute somente aplicativos pesados com:

```bash
prime-run PROGRAMA
```

Antes de alterar essa escolha, registre o estado atual e teste tanto a tela interna quanto as saídas externas. A conexão física das portas pode depender da GPU dedicada.

## Tela interna no Niri

O nome reportado pelo Niri é `eDP-1`. Um bloco mínimo para usar a resolução e a taxa nativas é:

```kdl
output "eDP-1" {
    mode "1920x1200@144.002"
    scale 1
}
```

Consulte os modos disponíveis em vez de copiar o valor literalmente:

```bash
niri msg outputs
```

O painel informa suporte a taxa de atualização variável. Ela pode ser habilitada no bloco da saída:

```kdl
output "eDP-1" {
    mode "1920x1200@144.002"
    scale 1
    variable-refresh-rate
}
```

Se houver oscilação de brilho, travamentos ou captura de tela inconsistente, remova `variable-refresh-rate` e recarregue a configuração antes de investigar outros componentes.

## Teclas e iluminação do teclado

O kernel expõe a iluminação como `asus::kbd_backlight`. Liste os níveis aceitos:

```bash
brightnessctl --device='asus::kbd_backlight' info
```

Exemplos para aumentar e reduzir um nível:

```bash
brightnessctl --device='asus::kbd_backlight' set +1
brightnessctl --device='asus::kbd_backlight' set 1-
```

Os módulos ASUS carregados automaticamente nesta unidade são:

```text
asus_wmi
asus_nb_wmi
asus_armoury
```

Confira com:

```bash
lsmod | grep '^asus'
```

Não foi necessário instalar `asusctl` ou `supergfxctl` para o funcionamento básico verificado aqui.

## Limite de carga da bateria

A bateria expõe `charge_control_end_threshold`. Confira o valor atual:

```bash
cat /sys/class/power_supply/BAT0/charge_control_end_threshold
```

Para limitar temporariamente a carga a 80%:

```bash
echo 80 | sudo tee /sys/class/power_supply/BAT0/charge_control_end_threshold
```

Para restaurar o limite de 100%:

```bash
echo 100 | sudo tee /sys/class/power_supply/BAT0/charge_control_end_threshold
```

O valor pode voltar ao padrão após reiniciar. Só crie uma regra persistente depois de testar se o limite escolhido é respeitado pelo firmware.

## Wi-Fi e áudio

O RTL8852BE funciona com o módulo incluído no kernel:

```bash
lspci -nnk | grep -A3 -i network
lsmod | grep rtw89
```

O áudio interno usa SOF. Em caso de dispositivo ausente, confirme firmware e perfil antes de adicionar opções de módulo:

```bash
pacman -Q sof-firmware alsa-ucm-conf
wpctl status
journalctl -b -k | grep -iE 'sof|snd|audio'
```

Não foi necessária uma correção específica para o Wi-Fi ou para o áudio no ambiente verificado.

## Limpeza da configuração herdada do Nitro 5

Foi encontrado este arquivo da instalação antiga:

```text
/etc/modprobe.d/blacklist-acer_wmi.conf
```

Ele contém `blacklist acer_wmi`. A regra não é necessária no ASUS e pode confundir diagnósticos futuros. Primeiro, confira o conteúdo:

```bash
cat /etc/modprobe.d/blacklist-acer_wmi.conf
```

Se o arquivo tiver apenas essa correção do Acer, remova-o e reconstrua a imagem de inicialização:

```bash
sudo rm /etc/modprobe.d/blacklist-acer_wmi.conf
sudo mkinitcpio -P
```

Procure outras referências ao hardware antigo antes de removê-las individualmente:

```bash
grep -RniE 'acer|nitro' \
  /etc/modprobe.d /etc/modules-load.d /etc/udev/rules.d \
  /etc/default/grub /etc/mkinitcpio.conf /etc/mkinitcpio.conf.d \
  2>/dev/null
```

Não remova um arquivo apenas por aparecer nessa busca: leia-o e confirme que foi criado especificamente para o Nitro 5.

## Checklist depois da migração

```bash
# Modelo e firmware
hostnamectl

# Drivers de vídeo e rede
lspci -nnk

# Saídas e modos reconhecidos pelo Niri
niri msg outputs

# GPU NVIDIA e versão do driver
nvidia-smi

# Módulos ASUS
lsmod | grep '^asus'

# Erros do kernel no boot atual
journalctl -b -k -p warning
```

Teste também suspensão e retomada, áudio, microfone, brilho, iluminação do teclado, Wi-Fi, Bluetooth, HDMI e USB-C. A presença de um dispositivo em `lspci` não garante que todos os seus estados de energia ou conexões externas estejam funcionando.

## Como desfazer

As consultas deste guia não modificam o sistema. Para desfazer os exemplos opcionais:

- restaure o limite da bateria para `100`;
- remova `variable-refresh-rate` do bloco de `eDP-1`;
- reverta apenas as variáveis de GPU que você próprio tiver alterado;
- restaure `blacklist-acer_wmi.conf` somente se a mesma instalação voltar a inicializar no Acer.

Depois de modificar opções de módulos ou a imagem de inicialização, execute novamente:

```bash
sudo mkinitcpio -P
```
