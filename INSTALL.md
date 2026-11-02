# Установка и обновление БезКода Агента

Выпуск: **0.2.1-rc.3.20261102.1**. Поставки x64: Windows Desktop (`.exe`), Ubuntu 22.04 Desktop (`.deb`), Windows-служба и Linux-служба (архивы `.tar.gz`). Службам не нужны системные Node, Python или pnpm.

## Desktop

Windows: закройте Агент, запустите новый EXE-установщик и выберите существующую установку. Настройки, сессии и рабочие области хранятся отдельно. Установщик не подписан, поэтому Windows может показывать неизвестного издателя.

Ubuntu 22.04 x64: скачайте `.deb` из релиза и установите `sudo apt install ./bezcoda-agent-0.2.1-rc.3.20261102.1-linux-amd64-unsigned.deb`. Для обновления закройте приложение. Если в прежней версии Координатор был встроен и его каталог находится внутри старого приложения, сначала сохраните профиль и выполните helper миграции из **новой распакованной поставки**, пока старое приложение доступно:

```sh
preflight=$(mktemp -d)
dpkg-deb --extract ./bezcoda-agent-0.2.1-rc.3.20261102.1-linux-amd64-unsigned.deb "$preflight"
node_bin=$(find "$preflight" -type f -path '*/resources/runtime/primary-runtime/dependencies/node/bin/node' -print -quit)
helper=$(find "$preflight" -type f -path '*/resources/*/node_modules/@bezcoda/agent/service/coordinator-migration.mjs' -print -quit)
test -n "$node_bin" && test -n "$helper"
"$node_bin" "$helper" --home "${BEZCODA_HOME:-${XDG_CONFIG_HOME:-$HOME/.config}/bezcoda-agent/dsh}"
sudo apt install ./bezcoda-agent-0.2.1-rc.3.20261102.1-linux-amd64-unsigned.deb
```

Не запускайте helper через sudo для пользовательского Desktop. При нестандартном каталоге данных передайте ваш фактический BEZCODA_HOME. Linux Desktop обновляется вручную.

## Linux-служба: новая установка

Распакуйте Linux service-архив в постоянный каталог, например `/opt/bezcoda-agent-releases/0.2.1-rc.3.20261102.1`. Из него:

```sh
sudo bash service/install.sh install --user bezcoda-agent --home /var/lib/bezcoda-agent --host 127.0.0.1 --port 3080
sudo bash service/install.sh start
sudo bash service/install.sh access
```

Последняя команда выводит ссылку с токеном. Храните её как пароль. Для reverse proxy передайте также `--public-url https://YOUR_DOMAIN` и соответствующие `--trusted-host`. Сохраняйте настроенные адрес, порт и параметры TLS при обновлении.

## Linux-служба: обновление с сохранением данных

Команды выполняет администратор (`sudo -i`). Ниже стандартное имя службы и корень данных. До остановки сверяйте их с `systemctl show bezcoda-agent -p User -p WorkingDirectory -p FragmentPath`.

1. Скачайте и проверьте архив; новая поставка должна лежать **рядом со старой**:

```sh
set -euo pipefail
umask 077
version=0.2.1-rc.3.20261102.1
base="https://github.com/olegpavlich/bezcoda-agent/releases/download/v$version"
work=$(mktemp -d /var/tmp/bezcoda-update.XXXXXX)
cd "$work"
archive="bezcoda-agent-$version-linux-x64-service.tar.gz"
curl --fail --location --output "$archive" "$base/$archive"
curl --fail --location --output SHA256SUMS.txt "$base/SHA256SUMS.txt"
awk -v file="$archive" '$2 == file' SHA256SUMS.txt > selected.sha256
test "$(wc -l < selected.sha256)" -eq 1
sha256sum --check selected.sha256
new="/opt/bezcoda-agent-releases/$version"
test ! -e "$new"
mkdir -p "$new"
tar -xzf "$archive" -C "$new"
test -x "$new/runtime/primary-runtime/dependencies/node/bin/node"
```

2. Остановите службу и сделайте согласованную приватную копию данных и регистрации:

```sh
backup="/root/bezcoda-backups/$(date -u +%Y%m%dT%H%M%SZ)"
mkdir -p "$backup"
chmod 700 "$backup"
cp -a /etc/systemd/system/bezcoda-agent.service "$backup/unit.service"
if [ -d /etc/systemd/system/bezcoda-agent.service.d ]; then
  cp -a /etc/systemd/system/bezcoda-agent.service.d "$backup/unit.service.d"
fi
systemctl stop bezcoda-agent
systemctl is-active --quiet bezcoda-agent && exit 1
tar --acls --xattrs -czpf "$backup/data.tar.gz" -C /var/lib bezcoda-agent
gzip -t "$backup/data.tar.gz"
```

Дополнительные рабочие каталоги вне `/var/lib/bezcoda-agent` копируются отдельно. Учетные данные находятся в резервной копии: не публикуйте её. Сохраните прежний runtime до завершения проверки. Если HOME служебного пользователя указывает на другой каталог, сохраните и его содержимое; установщик существующую учётную запись не пересоздаёт.

3. Зарегистрируйте новую поставку с **прежними сетевыми параметрами**, затем запустите. Пример для локального reverse proxy:

```sh
bash "$new/service/install.sh" install \
  --user bezcoda-agent --home /var/lib/bezcoda-agent \
  --host 127.0.0.1 --port 3080 \
  --public-url https://YOUR_DOMAIN --trusted-host YOUR_DOMAIN
bash "$new/service/install.sh" start
systemctl is-active --quiet bezcoda-agent
bash "$new/service/install.sh" access
```

Установщик сохраняет ранее установленный Координатор **до** смены регистрации службы. Чистая поставка содержит только обязательный профиль Агент; дополнительные архивы плагинов в этот релиз не входят. Откройте ссылку доступа, проверьте сессии, рабочие области, вход и установленные плагины. Не удаляйте прежний runtime и копию до проверки.

Если установка или запуск завершились ошибкой, служба может остаться остановленной. Исправьте причину либо выполните откат.

## Откат Linux-службы

Используйте значение `backup` из предыдущих команд. Восстанавливаются **и unit, и согласованная копия профиля**: миграция могла поменять ссылки плагинов. Старый runtime должен оставаться по исходному пути.

```sh
systemctl stop bezcoda-agent
failed="/var/lib/bezcoda-agent-failed-$(date -u +%Y%m%dT%H%M%SZ)"
test ! -e "$failed"
mv /var/lib/bezcoda-agent "$failed"
tar --acls --xattrs -xzpf "$backup/data.tar.gz" -C /var/lib
cp -a "$backup/unit.service" /etc/systemd/system/bezcoda-agent.service
systemctl daemon-reload
systemctl start bezcoda-agent
systemctl is-active --quiet bezcoda-agent
```

Каталог после неудачного обновления сохранён в `failed`. Если вы меняли drop-in настройки, восстановите также сохранённый `unit.service.d` перед `daemon-reload`.

## Windows-служба

Распакуйте Windows service-архив в постоянный каталог. В PowerShell администратора из этого каталога:

```powershell
.\service\install.ps1 install -DataRoot 'C:\ProgramData\bezcoda-agent' -ListenHost '127.0.0.1' -ListenPort 3080
.\service\install.ps1 start
.\service\install.ps1 access
```

Для обновления сохраните данные и регистрацию WinSW, остановите службу, распакуйте новую поставку отдельно и вызовите её установщик с теми же параметрами имени службы, данных и сети. Прежнюю поставку сохраняйте до успешной миграции Координатора и проверки запуска. Данные рабочей службы находятся в выбранном `DataRoot`, а не в каталоге архива. Точные имена параметров доступны в начале `service/install.ps1`.
