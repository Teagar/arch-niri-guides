# Rocksmith 2014 com M-VAVE Tank-G no Arch Linux

## Objetivo

Usar a pedaleira M-VAVE Tank-G como entrada de guitarra no Rocksmith 2014 Remastered, executado pela Steam/Proton, e os alto-falantes internos como saída no Arch Linux com PipeWire.

A solução usa:

- RS_ASIO para oferecer uma entrada ASIO ao Rocksmith;
- PipeASIO para levar essa entrada ao PipeWire;
- WASAPI/Wine-PipeWire para a saída do jogo;
- uma quirk do `snd-usb-audio` para o controlador Jieli da Tank-G;
- um serviço de usuário para refazer os links PipeWire após resets USB.

## Ambiente verificado

- Arch Linux, Niri e Wayland;
- kernel `7.2.8-arch1-2`;
- Steam nativo `1.0.0.87`;
- Proton 11, com new WoW64;
- PipeWire `1.6.9` e WirePlumber `0.5.18`;
- PipeASIO `1.10.0`;
- RS_ASIO `0.7.5`;
- Rocksmith 2014 Remastered, App ID `221680`;
- Tank-G identificada como `4c4a:c755 Jieli Technology USB Composite Device`.

O procedimento foi validado na calibração, no afinador e durante o tutorial. A Tank-G ainda pode reiniciar fisicamente no barramento USB, mas o serviço descrito abaixo restaura a entrada sem reiniciar o jogo.

## 1. Preparar o sistema

Instale os componentes de áudio necessários:

```bash
sudo pacman -S --needed \
  steam pipewire pipewire-audio pipewire-pulse wireplumber lib32-pipewire
```

Depois de uma atualização de kernel, confirme que o kernel em execução corresponde ao pacote instalado:

```bash
uname -r
pacman -Q linux
```

Se as versões forem diferentes, reinicie antes de continuar. Neste caso, a Tank-G não apareceu no ALSA enquanto o sistema executava um kernel antigo sem os módulos correspondentes.

Conecte a Tank-G e confirme a identificação:

```bash
lsusb | grep '4c4a:c755'
cat /proc/asound/cards
wpctl status -n
```

## 2. Instalar RS_ASIO e PipeASIO

Baixe o RS_ASIO na página oficial de releases:

- <https://github.com/mdias/rs_asio/releases>

Copie `avrt.dll`, `RS_ASIO.dll` e `RS_ASIO.ini` da versão `0.7.5` para:

```text
~/.local/share/Steam/steamapps/common/Rocksmith2014/
```

Baixe o PipeASIO `1.10.0` e seu gerenciador na página oficial:

- <https://github.com/M0n7y5/pipeasio/releases>

O Rocksmith é um aplicativo Windows de 32 bits. No PipeASIO Manager, selecione o prefixo Steam do Rocksmith (`221680`), instale também o front-end de 32 bits e execute a verificação. A instalação para Proton deve ficar em `~/.local`, pois o contêiner da Steam não enxerga a instalação Wine em `/usr`.

No Steam, abra **Rocksmith 2014 > Propriedades > Compatibilidade**, force **Proton 11** e inicie o jogo uma vez para criar o prefixo. Proton 9 não carregou `pipeasio32.so` neste ambiente.

Em **Propriedades > Geral > Opções de inicialização**, use um caminho absoluto e substitua `SEU_USUARIO`:

```bash
PROTON_USE_WOW64=1 WINEDLLPATH=/home/SEU_USUARIO/.local/lib/wine %command%
```

Não mantenha `PIPEWIRE_LATENCY` nessa linha; o PipeASIO controla o quantum pela própria configuração.

## 3. Aplicar a quirk do controlador Jieli

Durante streaming, o dispositivo apresentava:

```text
cannot get freq at ep 0x2
cannot get freq at ep 0x83
USB disconnect
device descriptor read/64, error -71
```

Crie a configuração persistente:

```bash
echo 'options snd_usb_audio quirk_flags=4c4a:c755:get_sample_rate' |
  sudo tee /etc/modprobe.d/mvave-tank-g.conf
```

Reinicie o computador. Depois, confirme:

```bash
cat /sys/module/snd_usb_audio/parameters/quirk_flags
```

O início deve conter:

```text
4c4a:c755:get_sample_rate
```

Essa quirk eliminou as mensagens `cannot get freq` e permitiu uma captura contínua de dez minutos. Ela não eliminou todos os resets físicos do firmware Jieli; o serviço da seção 7 contorna a perda dos links causada por esses resets.

## 4. Identificar a fonte da Tank-G

Liste as portas de saída do PipeWire:

```bash
pw-link -o | grep 'Jieli_Technology_USB_Composite_Device.*capture_FL'
```

O resultado terá esta forma:

```text
alsa_input.usb-Jieli_Technology_USB_Composite_Device_SERIAL-00.analog-stereo:capture_FL
```

Remova o sufixo `:capture_FL`. O restante é o nome usado em `input_device` na próxima seção.

Defina também os alto-falantes internos como saída padrão. Localize o ID em `wpctl status -n` e execute:

```bash
wpctl set-default ID_DA_SAIDA_INTERNA
```

## 5. Configurar PipeASIO

Crie `~/.config/pipeasio/config.ini` e substitua o serial pelo valor obtido acima:

```ini
[pipeasio]
inputs=2
outputs=2
auto_connect=1
input_device=alsa_input.usb-Jieli_Technology_USB_Composite_Device_SERIAL-00.analog-stereo
output_device=alsa_output.pci-0000_00_1f.3.analog-stereo
sample_rate=0
fixed_buffer_size=1
buffer_size=512
follow_device_clock=0
realtime=0
```

O `output_device` acima é o nome do dispositivo interno no equipamento testado. Descubra o nome correto no seu sistema com `wpctl status -n`.

A Tank-G anuncia 44,1 kHz, enquanto o Rocksmith exige 48 kHz. `sample_rate=0` permite que o Rocksmith escolha 48 kHz e que o PipeWire faça o resampling. O buffer 512 foi o melhor compromisso estável neste teste.

## 6. Configurar Rocksmith e RS_ASIO

Em `Rocksmith.ini`, mantenha estas opções na seção `[Audio]`:

```ini
[Audio]
EnableMicrophone=0
ExclusiveMode=1
LatencyBuffer=2
ForceDefaultPlaybackDevice=
ForceWDM=0
ForceDirectXSink=0
DumpAudioLog=0
MaxOutputBufferSize=0
RealToneCableOnly=1
MonoToStereoChannel=0
Win32UltraLowLatencyMode=1
```

Em `RS_ASIO.ini`, use PipeASIO somente para a entrada e WASAPI para a saída:

```ini
[Config]
EnableWasapiOutputs=1
EnableWasapiInputs=0
EnableAsio=1

[Asio]
BufferSizeMode=driver
CustomBufferSize=

[Asio.Output]
Driver=
BaseChannel=0
AltBaseChannel=
EnableSoftwareEndpointVolumeControl=1
EnableSoftwareMasterVolumeControl=1
SoftwareMasterVolumePercent=100
EnableRefCountHack=

[Asio.Input.0]
Driver=PipeASIO
Channel=0
EnableSoftwareEndpointVolumeControl=1
EnableSoftwareMasterVolumeControl=1
SoftwareMasterVolumePercent=200
EnableRefCountHack=

[Asio.Input.1]
Driver=
Channel=1
EnableSoftwareEndpointVolumeControl=1
EnableSoftwareMasterVolumeControl=1
SoftwareMasterVolumePercent=100
EnableRefCountHack=

[Asio.Input.Mic]
Driver=
Channel=1
EnableSoftwareEndpointVolumeControl=1
EnableSoftwareMasterVolumeControl=1
SoftwareMasterVolumePercent=100
EnableRefCountHack=
```

Pontos importantes:

- `RealToneCableOnly=1` é necessário para esta entrada simulada como Real Tone Cable;
- os controles de volume por software devem permanecer habilitados, pois desabilitá-los fez a calibração retornar `E_FAIL`;
- `SoftwareMasterVolumePercent=200` acrescenta cerca de 6 dB ao sinal baixo da Tank-G;
- não aumente a fonte PipeWire acima de 100%: 200% produziu aproximadamente +18 dB e muito ruído;
- não use PipeASIO também como saída neste conjunto de dispositivos: agregar o relógio USB de 44,1 kHz à saída interna de 48 kHz deixou música, vozes e menus robotizados. A saída WASAPI/Wine-PipeWire ficou limpa.

## 7. Restaurar os links após resets USB

O firmware Jieli pode desconectar e enumerar novamente a Tank-G durante uma sessão. O WirePlumber recria o dispositivo, mas o PipeASIO mantém portas sem links. O jogo continua aberto e a entrada volta assim que as novas portas são ligadas.

Crie `~/.local/bin/rocksmith-tank-g-relink`:

```bash
#!/usr/bin/env bash

set -uo pipefail

relink() {
  local source_fl source_fr

  source_fl="$(pw-link -o 2>/dev/null |
    grep -m1 -E '^alsa_input\.usb-Jieli_Technology_USB_Composite_Device_.*:capture_FL$')"
  [[ -n "$source_fl" ]] || return 0

  source_fr="${source_fl%:capture_FL}:capture_FR"
  pw-link "$source_fl" 'Rocksmith2014:in_1' >/dev/null 2>&1 || true
  pw-link "$source_fr" 'Rocksmith2014:in_2' >/dev/null 2>&1 || true
}

while true; do
  relink

  # Repete após mudanças no grafo e reconecta o monitor se o PipeWire reiniciar.
  pw-link -lm 2>/dev/null | while IFS= read -r _; do
    relink
  done

  sleep 1
done
```

Torne-o executável:

```bash
chmod +x ~/.local/bin/rocksmith-tank-g-relink
```

Crie `~/.config/systemd/user/rocksmith-tank-g-relink.service`:

```ini
[Unit]
Description=Restore Tank-G links after USB reconnects
After=pipewire.service wireplumber.service

[Service]
Type=simple
ExecStart=%h/.local/bin/rocksmith-tank-g-relink
Restart=always
RestartSec=1

[Install]
WantedBy=default.target
```

Ative o serviço:

```bash
systemctl --user daemon-reload
systemctl --user enable --now rocksmith-tank-g-relink.service
systemctl --user status rocksmith-tank-g-relink.service
```

O script identifica a fonte pelo fabricante e modelo, sem depender do serial da unidade. No teste, um link removido manualmente foi restaurado em menos de dois segundos e a guitarra voltou automaticamente após um reset real durante o tutorial.

## 8. Validar

Abra o Rocksmith pela Steam e faça, nesta ordem:

1. calibração;
2. afinador;
3. tutorial ou uma música por pelo menos 15 minutos.

Confirme o carregamento do driver:

```bash
grep -E 'Wrapper DLL loaded|PipeASIO|actual buffer duration|ERROR' \
  ~/.local/share/Steam/steamapps/common/Rocksmith2014/RS_ASIO-log.txt
```

O log deve mostrar PipeASIO e um buffer de aproximadamente `10ms (512 frames)`.

Veja os links enquanto o jogo está aberto:

```bash
pw-link -l | grep -A 3 -B 2 'Rocksmith2014:in_'
```

Diagnostique resets USB com:

```bash
journalctl -k -b --no-pager |
  grep -i -E 'usb.*disconnect|error -71|cannot get freq|Jieli'
```

Um reset ainda pode causar uma interrupção curta. O estado esperado é a entrada voltar em um a três segundos sem reiniciar o Rocksmith.

## Solução de problemas

### `cannot load pipeasio32.so`

- confirme Proton 11;
- confirme `PROTON_USE_WOW64=1`;
- use o caminho absoluto em `WINEDLLPATH`;
- confirme que o PipeASIO foi instalado no prefixo `221680` com o componente de 32 bits.

### Música, voz e menus robotizados

Confirme em `RS_ASIO.ini`:

```ini
EnableWasapiOutputs=1

[Asio.Output]
Driver=
```

Usar a saída PipeASIO junto da entrada Tank-G agregou relógios de 44,1 e 48 kHz e gerou xruns. Aumentar o buffer até 1024 não corrigiu esse caso.

### Calibração presa em “KEEP GOING”

- confirme `RealToneCableOnly=1`;
- mantenha os dois controles de volume por software da entrada em `1`;
- use ganho inicial de 200%;
- confira se os links `capture_FL -> in_1` e `capture_FR -> in_2` existem.

### A guitarra para, mas o restante do áudio continua

Confira o serviço e o kernel:

```bash
systemctl --user status rocksmith-tank-g-relink.service
journalctl --user -u rocksmith-tank-g-relink.service -b
journalctl -k -b --no-pager | grep -i -E 'USB disconnect|error -71'
```

## Como desfazer

Desative a recuperação automática:

```bash
systemctl --user disable --now rocksmith-tank-g-relink.service
rm ~/.config/systemd/user/rocksmith-tank-g-relink.service
rm ~/.local/bin/rocksmith-tank-g-relink
systemctl --user daemon-reload
```

Remova a quirk e reinicie:

```bash
sudo rm /etc/modprobe.d/mvave-tank-g.conf
sudo reboot
```

Para remover RS_ASIO, apague somente `avrt.dll`, `RS_ASIO.dll`, `RS_ASIO.ini` e `RS_ASIO-log.txt` da pasta do Rocksmith. Use **Remove** no PipeASIO Manager para desfazer a instalação no prefixo sem alterar outros componentes do Proton.
