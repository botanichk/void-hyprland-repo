# Исправление звука на Void Linux + Hyprland

Проблема: PulseAudio конфликтует с PipeWire. Звук не работает, `pactl info` отвечает "Connection timed out".

## Алгоритм исправления

### Шаг 1. Удалить runit сервис PulseAudio

```bash
sudo rm /var/service/pulseaudio
```

Это главная причина конфликта. Runit перезапускал PulseAudio и занимал сокет.

### Шаг 2. Отключить PulseAudio autostart

```bash
mkdir -p ~/.config/autostart
cp /etc/xdg/autostart/pulseaudio.desktop ~/.config/autostart/pulseaudio.desktop
```

Добавить в конец файла строку:

```
Hidden=true
```

### Шаг 3. Создать wrapper для запуска аудио

Создать файл `~/.config/start-audio.sh`:

```bash
#!/bin/bash

# Ждём D-Bus
for i in $(seq 1 10); do
    if [ -S "/run/user/1000/bus" ]; then
        break
    fi
    sleep 0.5
done

# Проверяем D-Bus
if [ -z "$DBUS_SESSION_BUS_ADDRESS" ]; then
    export DBUS_SESSION_BUS_ADDRESS="unix:path=/run/user/1000/bus"
fi

# Запускаем PipeWire если не запущен
if ! pgrep -x pipewire > /dev/null; then
    /usr/bin/pipewire &
    sleep 1
fi

# Запускаем WirePlumber если не запущен
if ! pgrep -x wireplumber > /dev/null; then
    /usr/bin/wireplumber &
    sleep 1
fi

# Запускаем PipeWire-Pulse если не запущен
if ! pgrep -f "pipewire.*pipewire-pulse.conf" > /dev/null; then
    /usr/bin/pipewire -c /usr/share/pipewire/pipewire-pulse.conf &
fi
```

Сделать исполняемым:

```bash
chmod +x ~/.config/start-audio.sh
```

### Шаг 4. Добавить автозапуск в Hyprland

Добавить в `~/.config/hypr/hyprland.conf`:

```
exec-once = sleep 1 && /home/ig_ro/.config/start-audio.sh
```

### Шаг 5. Перезагрузиться

```bash
sudo reboot
```

### Шаг 6. Проверить звук

```bash
pactl info
pactl list sinks short
```

Если HDMI не по умолчанию:

```bash
pactl set-default-sink alsa_output.pci-0000_01_00.1.hdmi-stereo
```

Проверить громкость:

```bash
pactl set-sink-volume @DEFAULT_SINK@ 100%
```

---

## Если звук пропал снова

1. Проверить наличие runit сервиса:
```bash
ls /var/service/ | grep pulse
```
Если есть — удалить: `sudo rm /var/service/pulseaudio`

2. Проверить D-Bus:
```bash
echo $DBUS_SESSION_BUS_ADDRESS
```
Если пусто — перезайти в Hyprland

3. Перезапустить PipeWire:
```bash
pkill -u ig_ro pipewire; pkill -u ig_ro wireplumber
rm -f /run/user/1000/pulse/native /run/user/1000/pulse/pid
```
Затем перезайти в сессию

---

## Важные замечания

- На Void Linux использовать `pkill` вместо `killall` (killall не установлен)
- PipeWire требует D-Bus session — запускать только из графической сессии
- `fuser` не установлен — использовать `ss -lx | grep pulse` для проверки сокетов
- Stale pulse socket нужно удалять перед перезапуском PipeWire-Pulse

## Конфигурационные файлы

- `~/.config/hypr/hyprland.conf` — основной конфиг Hyprland
- `~/.config/start-audio.sh` — wrapper для запуска D-Bus + PipeWire
- `~/.config/autostart/pulseaudio.desktop` — отключен PulseAudio
- `~/.config/sound-restore.sh` — восстановление звука через HDMI

## Оборудование

- Intel Comet Lake (H510) + NVIDIA RTX 2060
- Realtek ALC897 (Intel HDA PCH) + NVIDIA HDMI audio
- Void Linux (rolling), Hyprland (Wayland)
