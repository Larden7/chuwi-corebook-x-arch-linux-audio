# CHUWI CoreBook X на Arch Linux: работает только один динамик, слышен только левый канал

[English version](README.en.md)

**Симптомы для поиска:** CHUWI CoreBook X, Arch Linux, PipeWire, Realtek ALC269VC,
работает только один динамик, звук идёт только из одного динамика, слышен только
левый канал, правый динамик не работает, нет стерео, в Windows 11 звук работает
нормально, `line_outs=1`, HDA pin `0x1b`.

## Проблема

На ноутбуке **CHUWI CoreBook X** после установки Arch Linux звук был только из
одного динамика: слышался только левый выход из правого динамика, второй динамик
молчал. В Windows 11 оба динамика работали нормально.

Баланс и громкость были в порядке:

- в PipeWire оба канала `front-left`/`front-right` на 100%;
- баланс `0.00`;
- `Speaker`, `PCM` и `Master` в ALSA не были приглушены;
- `speaker-test` формально проигрывал оба канала, но физически был слышен только
  один динамик.

То есть это не настройка баланса и не проблема приложения. Причина была на
уровне описания аудиокодека.

## Оборудование и стенд

| Параметр | Значение |
|---|---|
| Ноутбук | CHUWI CoreBook X |
| PCI-аудио | Intel Ice Lake-LP Smart Sound Technology, `8086:34c8` |
| PCI SSID | `1e50:7036` |
| Кодек | Realtek ALC269VC, `10ec:0269` |
| Subsystem ID кодека | `1e50:7036` |
| ОС | Arch Linux |
| Ядро | `7.2.9-arch1-1` |
| Звуковой сервер | PipeWire 1.6.9 + WirePlumber, PulseAudio-совместимый сервер |

Проверить свои идентификаторы:

```bash
lspci -nnk | grep -A4 -i audio
cat /proc/asound/cards
grep -E 'Codec:|Subsystem Id:|Vendor Id:' /proc/asound/card0/codec#0
```

## Причина

BIOS ноутбука отдаёт для второго внутреннего выхода (`pin 0x1b`) состояние
«не подключён» (`0x411111f0`). Поэтому Linux при автонастройке видел только один
выход на динамики:

```text
autoconfig for ALC269VC: line_outs=1 (0x14/0x0/0x0/0x0/0x0) type:speaker
```

Второй выход `0x1b`, к которому подключён второй динамик, оставался выключенным.
Драйвер Windows знает особенность этой платы и включает его, а стандартная
автонастройка Linux — нет.

В актуальном исходнике ядра Linux для этого же SSID уже есть quirk:
`1e50:7036` → `ALC233_FIXUP_WUJIE_SPEAKERS`, который назначает `0x1b` как
внутренний динамик:

```c
SND_PCI_QUIRK(0x1e50, 0x7036, "Acer Gadget E10 ETBook",
              ALC233_FIXUP_WUJIE_SPEAKERS),
```

Ссылка на исходник:
<https://github.com/torvalds/linux/blob/master/sound/hda/codecs/realtek/alc269.c>

Но в используемом ядре `7.2.9-arch1-1` этот quirk ещё не применился
автоматически, поэтому нужен ручной pin override.

## Решение

Нужно переопределить default pin configuration для HDA-узла `0x1b`:

```text
0x1b: 0x90170150
```

Это значит: fixed internal speaker, analog, association 5. Такое же значение
использует upstream-исправление для этого SSID.

### 1. Создать early firmware patch

```bash
sudo tee /lib/firmware/hda-jack-retask.fw > /dev/null <<'EOF'
[codec]
0x10ec0269 0x1e507036 0

[pincfg]
0x1b 0x90170150
EOF
```

### 2. Подключить patch к модулю `snd-hda-intel`

```bash
echo 'options snd-hda-intel patch=hda-jack-retask.fw' | \
  sudo tee /etc/modprobe.d/hda-jack-retask.conf
```

### 3. Перезагрузиться

```bash
reboot
```

Патч применяется при загрузке драйвера, до запуска PipeWire. Это важно: попытка
применить настройку через `/sys/class/sound/card0/hwC0D0/reconfig` на работающей
системе часто заканчивается ошибкой:

```text
The codec is being used, can't reconfigure.
```

Останавливать `pipewire`, `wireplumber` и `pipewire-pulse` недостаточно:
сокеты автозапуска могут поднять их снова, а кодек всё равно останется занятым.
Early firmware patch надёжнее и сохраняется между загрузками.

## Проверка

После перезагрузки:

```bash
journalctl -k -b --no-pager | grep -E 'autoconfig for ALC269|speaker_outs'
```

Ожидаемое изменение:

```text
autoconfig for ALC269VC: line_outs=2 (0x14/0x1b/0x0/0x0/0x0) type:speaker
```

Также должен появиться второй регулятор динамика:

```bash
amixer -c 0 scontents | grep -A5 -i 'Bass Speaker'
```

Проверить каналы:

```bash
speaker-test -D default -c 2 -t wav -l 1
```

После патча обязательно проверьте громкость в `alsamixer`/`pavucontrol`:
`Speaker`, `Bass Speaker`, `PCM` и громкость активного sink не должны быть
на нуле или в mute.

В проверенной системе после перезагрузки ядро показало:

```text
autoconfig for ALC269VC: line_outs=2 (0x14/0x1b/0x0/0x0/0x0) type:speaker
```

а в кодеке появился второй выход:

```text
Node 0x1b [Pin Complex]
  Control: name="Bass Speaker Playback Switch", index=0, device=0
  Pin-ctls: 0x40: OUT VREF_HIZ
```

При этом `Pin Default` в `/proc/asound/card0/codec#0` всё ещё может
показывать старое `0x411111f0`: ориентируйтесь на `line_outs=2`, наличие
`Bass Speaker` и реальный тест каналов.

## Если звука всё ещё нет

1. Убедитесь, что после перезагрузки в журнале ядра именно `line_outs=2` с
   `0x14/0x1b`, а не `line_outs=1`.
2. Проверьте, что patch реально загружен:

   ```bash
   cat /sys/module/snd_hda_intel/parameters/patch
   ```

   Должно быть указано `hda-jack-retask.fw`.
3. Проверьте `alsamixer -c 0`: `Speaker` и `Bass Speaker` не на нуле, не в mute.
4. Проверьте активный профиль и порт:

   ```bash
   pactl list sinks
   ```

   Должен быть активен `analog-output-speaker`; профиль — `analog-stereo`.
5. Перезапустите пользовательские службы звука:

   ```bash
   systemctl --user restart wireplumber pipewire pipewire-pulse
   ```

## Откат

Удалить оба файла и перезагрузиться:

```bash
sudo rm -f /etc/modprobe.d/hda-jack-retask.conf \
           /lib/firmware/hda-jack-retask.fw
sudo reboot
```

## Альтернатива: дождаться исправления в ядре

Upstream quirk для `1e50:7036` уже есть в актуальном mainline. Когда он придёт
в ядро вашего дистрибутива, ручной патч можно будет убрать. Проверять по той же
строке `line_outs=2` в журнале после обновления ядра.

## Файлы

- [`hda-jack-retask.fw`](hda-jack-retask.fw) — early firmware patch;
- [`hda-jack-retask.conf`](hda-jack-retask.conf) — параметр модуля.

## Полезные ссылки

- [ALSA: HDA — pin configuration](https://www.kernel.org/doc/html/latest/sound/hd-audio/notes.html)
- [Arch Wiki: Advanced Linux Sound Architecture](https://wiki.archlinux.org/title/Advanced_Linux_Sound_Architecture)
- [Исходник Realtek ALC269 в ядре Linux](https://github.com/torvalds/linux/blob/master/sound/hda/codecs/realtek/alc269.c)
