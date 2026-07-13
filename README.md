# Исправление звука на Void Linux + Hyprland

Проблема: PulseAudio конфликтует с PipeWire. Звук не работает, `pactl info` отвечает "Connection timed out" или "Connection refused".

## Алгоритм исправления

### Шаг 1. Удалить runit сервис PulseAudio

```bash
sudo rm /var/service/pulseaudio
```

Это главная причина конфликта. Runit перезапускал PulseAudio и занимал сокет.

### Шаг 2. Заблокировать autospawn PulseAudio

Создать `~/.config/pulse/client.conf`:

```ini
autospawn=no
daemon-binary=/bin/false
```

Без этого pulseaudio будет автоматически запускаться при любом обращении к аудио (например `pactl info`) и захватывать сокет.

### Шаг 3. Удалить WirePlumber user config (если есть)

```bash
rm -f ~/.config/wireplumber/wireplumber.conf
```

Кастомный конфиг часто не содержит нужных компонентов. Системный (`/usr/share/wireplumber/wireplumber.conf`) работает корректно.

### Шаг 4. Удалить Portal конфиг PipeWire (если есть)

```bash
rm -f ~/.config/pipewire/pipewire.conf.d/10-portal.conf
```

Модуль Portal может мешать работе pipewire-pulse.

### Шаг 5. Создать wrapper для запуска аудио

Создать файл `~/.config/start-audio.sh`:

```bash
#!/bin/bash
# Автозапуск аудио-стека: D-Bus + PipeWire
# Вызывается из Hyprland exec-once

export DBUS_SESSION_BUS_ADDRESS="unix:path=/run/user/1000/bus"
export XDG_RUNTIME_DIR=/run/user/1000

# 0. Убиваем настоящий PulseAudio если запущен — он конфликтует с PipeWire
if pgrep -x pulseaudio >/dev/null 2>&1; then
    pkill pulseaudio
    sleep 1
fi
# Чистим pid-файл pulseaudio, иначе pipewire-pulse не создаст свой сокет
rm -f /run/user/1000/pulse/pid

# 1. Запускаем D-Bus если не запущен
dbus_alive() {
    [ -S /run/user/1000/bus ] || return 1
    timeout 1 dbus-send --session --dest=org.freedesktop.DBus --type=method_call --print-reply \
        /org/freedesktop/DBus org.freedesktop.DBus.ListNames >/dev/null 2>&1
}
if ! dbus_alive; then
    rm -f /run/user/1000/bus
    dbus-daemon --session --address="unix:path=/run/user/1000/bus" --nofork &
    for i in $(seq 1 10); do
        dbus_alive && break
        sleep 0.5
    done
fi

# 2. PipeWire
if ! pgrep -x pipewire >/dev/null 2>&1; then
    /usr/bin/pipewire &
    sleep 1
fi

# 3. WirePlumber
if ! pgrep -x wireplumber >/dev/null 2>&1; then
    /usr/bin/wireplumber &
    sleep 1
fi

# 4. PipeWire-Pulse
if ! pgrep -f "pipewire.*pipewire-pulse.conf" >/dev/null 2>&1; then
    /usr/bin/pipewire -c /usr/share/pipewire/pipewire-pulse.conf &
    sleep 1
fi

# 5. Ждём PulseAudio-совместимый сервер
for i in $(seq 1 10); do
    if pactl info >/dev/null 2>&1; then
        break
    fi
    sleep 1
done

# 6. Ставим HDMI по умолчанию
pactl set-default-sink alsa_output.pci-0000_01_00.1.hdmi-stereo 2>/dev/null
pactl set-sink-volume alsa_output.pci-0000_01_00.1.hdmi-stereo 100% 2>/dev/null
pactl set-sink-mute alsa_output.pci-0000_01_00.1.hdmi-stereo 0 2>/dev/null
```

Сделать исполняемым:

```bash
chmod +x ~/.config/start-audio.sh
```

### Шаг 6. Добавить автозапуск в Hyprland

Добавить в `~/.config/hypr/hyprland.conf` или `~/.config/hypr/hyprland.lua`:

```
exec-once = /home/ig_ro/.config/start-audio.sh
```

И установить PULSE_SERVER:

```
env = PULSE_SERVER,unix:/run/user/1000/pulse/native
```

### Шаг 7. Перезагрузиться

```bash
sudo reboot
```

### Шаг 8. Проверить звук

```bash
pactl info
pactl list sinks short
```

Должно показать:
```
Сервер: PulseAudio (on PipeWire 1.6.7)
```

Если HDMI не по умолчанию:

```bash
pactl set-default-sink alsa_output.pci-0000_01_00.1.hdmi-stereo
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
pkill -9 -u ig_ro wireplumber "pipewire.*pulse"
rm -f /run/user/1000/pulse/native /run/user/1000/pulse/pid
```
Затем перезайти в сессию

---

## Важные замечания

- **stale pid-файл** `/run/user/1000/pulse/pid` от pulseaudio блокирует pipewire-pulse — всегда удалять при перезапуске
- **`~/.config/pulse/client.conf`** с `autospawn=no` и `daemon-binary=/bin/false` — критически важен для блокировки autospawn
- **WirePlumber user config** (`~/.config/wireplumber/wireplumber.conf`) должен содержать компонент `policy.client.access`, иначе WirePlumber не стартует. Проще удалить и использовать системный
- **PipeWire Portal модуль** (`module.portal = true`) может мешать — лучше не включать
- **`pactl info`** может виснуть внутри Flatpak песочницы (pipewire-pulse + user namespace issue), но `paplay` (Simple API) работает
- На Void Linux использовать `pkill` вместо `killall` (killall не установлен)
- `fuser` не установлен — использовать `ss -lx | grep pulse` для проверки сокетов

## Конфигурационные файлы

- `~/.config/hypr/hyprland.conf` — основной конфиг Hyprland
- `~/.config/start-audio.sh` — wrapper для запуска D-Bus + PipeWire
- `~/.config/pulse/client.conf` — блокировка автозапуска PulseAudio
- `~/.config/sound-restore.sh` — восстановление звука через HDMI

## Оборудование

- Intel Comet Lake (H510) + NVIDIA RTX 2060
- Realtek ALC897 (Intel HDA PCH) + NVIDIA HDMI audio
- Void Linux (rolling), Hyprland (Wayland)
