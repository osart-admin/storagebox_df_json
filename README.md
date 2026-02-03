# Template Hetzner StorageBox

Zabbix шаблон для мониторинга Hetzner Storage Box через SFTP.

## Описание

Шаблон собирает данные об использовании дискового пространства Hetzner Storage Box с помощью команды `df -h` через SFTP-соединение. Использует внешний скрипт и зависимые элементы данных с парсингом JSON.

## Требования

- Zabbix Server 7.x
- Установленный `sshpass` на сервере Zabbix
- Внешний скрипт `storagebox_df_json.sh` в директории ExternalScripts

## Установка

1. Скопируйте скрипт `storagebox_df_json.sh` в директорию ExternalScripts Zabbix сервера
2. Сделайте скрипт исполняемым: `chmod +x storagebox_df_json.sh`
3. Импортируйте шаблон `Template_Hetzner_StorageBox.yaml` в Zabbix
4. Создайте хост и привяжите шаблон
5. Настройте макросы хоста

## Макросы

| Макрос | Обязательный | По умолчанию | Описание |
|--------|--------------|--------------|----------|
| `{$SB_HOST}` | Да | - | Хост StorageBox (например: u12345.your-storagebox.de) |
| `{$SB_USER}` | Да | - | Имя пользователя (например: u12345) |
| `{$SB_PASS}` | Да | - | Пароль (секретный текст) |
| `{$SB_PORT}` | Нет | 22 | SFTP порт |
| `{$SB_WARN}` | Нет | 85 | Порог предупреждения (%) |
| `{$SB_HIGH}` | Нет | 95 | Критический порог (%) |

## Элементы данных (Items)

| Имя | Ключ | Тип | Описание |
|-----|------|-----|----------|
| DF JSON (master) | `storagebox_df_json.sh[...]` | External | Мастер-элемент, возвращает JSON |
| OK | `storagebox.ok` | Dependent | Статус сбора (1=ОК, 0=ошибка) |
| Error | `storagebox.error` | Dependent | Текст ошибки |
| Used percent | `storagebox.pused` | Dependent | Процент использования (%) |
| Used | `storagebox.used_bytes` | Dependent | Использовано (байты) |
| Free | `storagebox.free_bytes` | Dependent | Свободно (байты) |
| Total | `storagebox.total_bytes` | Dependent | Всего (байты) |

## Триггеры

| Имя | Severity | Условие |
|-----|----------|---------|
| Collector error | Average | `storagebox.ok = 0` |
| Usage > {$SB_WARN}% | Warning | `storagebox.pused > 85%` |
| Usage > {$SB_HIGH}% | High | `storagebox.pused > 95%` |

## Графики

### StorageBox: Used %
- Показывает процент использования
- Красная линия с градиентом
- Ось Y: фиксированная шкала 0-100%

### StorageBox: Capacity
- Total — синяя линия (общий объём)
- Used — зелёная заливка (использовано)
- Ось Y: от 0 до полного объёма диска (Total)

## Теги элементов

Все элементы имеют теги для фильтрации:
- `component: storage`
- `vendor: hetzner`
- `service: storagebox`
- `class: capacity` / `status` / `raw`

## Интервал сбора

Данные собираются каждые **10 минут**.

## Внешний скрипт

Создайте файл `/usr/lib/zabbix/externalscripts/storagebox_df_json.sh`:

```bash
#!/usr/bin/env bash
set -euo pipefail

# Hetzner Storage Box disk usage via SFTP `df -h` (password auth).
# Real output example:
#     Size     Used    Avail   (root)    %Capacity
#    1.0TB    744GB    280GB    280GB          72%
#
# Args:
#  1: user (e.g., u454424)
#  2: host (e.g., u454424.your-storagebox.de)
#  3: port (default 22)
#  4: path (kept for compatibility; not used by df -h output here) default /
#  5: password (required)

SB_USER="${1:-}"
SB_HOST="${2:-}"
SB_PORT="${3:-22}"
SB_PATH="${4:-/}"      # kept for interface compatibility; not used by df -h output here
SB_PASS="${5:-}"

json_error() {
  local err="$1"
  local raw="${2:-}"
  raw="$(echo "$raw" | head -n 80 | tr -d '\r' | sed 's/"/\\"/g')"
  echo "{\"ok\":0,\"error\":\"$err\",\"raw\":\"$raw\"}"
}

if [[ -z "$SB_USER" || -z "$SB_HOST" ]]; then
  json_error "missing user/host"
  exit 0
fi

if [[ -z "$SB_PASS" ]]; then
  json_error "missing password"
  exit 0
fi

if ! command -v sshpass >/dev/null 2>&1; then
  json_error "sshpass not installed"
  exit 0
fi

# Use env to avoid exposing password in argv.
export SSHPASS="$SB_PASS"

# Separate known_hosts to avoid permission issues with zabbix user's home.
KNOWN="/tmp/zbx_storagebox_known_hosts"

# Run SFTP batch: df -h is supported by Hetzner StorageBox.
raw="$(
  sshpass -e sftp -q -P "$SB_PORT" \
    -oBatchMode=no \
    -oConnectTimeout=10 \
    -oUserKnownHostsFile="$KNOWN" \
    -oStrictHostKeyChecking=accept-new \
    "${SB_USER}@${SB_HOST}" <<'EOF' 2>&1 || true
df -h
quit
EOF
)"

# Find first line that ends with NN% (data line)
data_line="$(echo "$raw" | awk 'NF>=4 && $NF ~ /^[0-9]+%$/ {print; exit}')"

if [[ -z "${data_line:-}" ]]; then
  json_error "cannot_parse_df" "$raw"
  exit 0
fi

# Expected: Size Used Avail (root) NN%
size="$(echo "$data_line" | awk '{print $1}')"
used="$(echo "$data_line" | awk '{print $2}')"
avail="$(echo "$data_line" | awk '{print $3}')"
pused="$(echo "$data_line" | awk '{print $NF}' | sed 's/%//')"

# Convert human sizes (K/M/G/T/P/E with optional B) to MB (base 1024).
to_mb() {
  local v="$1"
  awk -v v="$v" '
    function pow1024(n, r){r=1; while(n-->0) r*=1024; return r}
    BEGIN{
      gsub(/,/,".",v)

      if (v ~ /^[0-9.]+([Kk][Bb]?|[Kk])$/){sub(/([Kk][Bb]?|[Kk])$/,"",v); printf "%.0f", v/1024; exit}
      if (v ~ /^[0-9.]+([Mm][Bb]?|[Mm])$/){sub(/([Mm][Bb]?|[Mm])$/,"",v); printf "%.0f", v; exit}
      if (v ~ /^[0-9.]+([Gg][Bb]?|[Gg])$/){sub(/([Gg][Bb]?|[Gg])$/,"",v); printf "%.0f", v*1024; exit}
      if (v ~ /^[0-9.]+([Tt][Bb]?|[Tt])$/){sub(/([Tt][Bb]?|[Tt])$/,"",v); printf "%.0f", v*pow1024(2); exit}
      if (v ~ /^[0-9.]+([Pp][Bb]?|[Pp])$/){sub(/([Pp][Bb]?|[Pp])$/,"",v); printf "%.0f", v*pow1024(3); exit}
      if (v ~ /^[0-9.]+([Ee][Bb]?|[Ee])$/){sub(/([Ee][Bb]?|[Ee])$/,"",v); printf "%.0f", v*pow1024(4); exit}

      if (v ~ /^[0-9.]+$/){printf "%.0f", v; exit}

      printf "0"
    }'
}

total_mb="$(to_mb "$size")"
used_mb="$(to_mb "$used")"
free_mb="$(to_mb "$avail")"

if [[ "$total_mb" -le 0 || "$used_mb" -lt 0 || "$free_mb" -lt 0 || -z "$pused" ]]; then
  json_error "parse_values_failed" "$raw"
  exit 0
fi

echo "{\"ok\":1,\"error\":\"\",\"total_mb\":$total_mb,\"used_mb\":$used_mb,\"free_mb\":$free_mb,\"pused\":$pused}"
```

После создания сделайте скрипт исполняемым:

```bash
chmod +x /usr/lib/zabbix/externalscripts/storagebox_df_json.sh
```

### Формат вывода скрипта

**Успех:**
```json
{"ok":1,"error":"","total_mb":1048576,"used_mb":761856,"free_mb":286720,"pused":72}
```

**Ошибка:**
```json
{"ok":0,"error":"missing password","raw":""}
```

## Автор

Custom template v3.0
