## Routerich: Настройка AmneziaWG и youtubeUnblock

Настроим **[Routerich AX3000](https://routerich.ru/products/ax3000)** так, чтобы на разных WiFi-сетях использовались разные способы обхода блокировок. Это избавит от необходимости устанавливать VPN-клиенты на каждое устройство.

**Характеристики настраиваемого роутера:**

- Модель: `Routerich AX3000 v1`
- Архитектура: `ARMv8 Processor rev 4`
- Целевая платформа: `mediatek/filogic`
- Версия встроенного ПО: `RouteRich 24.10.5 r29087-d9c5716d1d RR-3.9.0 / LuCI openwrt-24.10 branch 26.039.68875~ec3d818`
- Версия ядра: `6.6.119`

**В результате настройки мы получим:**

При подключении по `WiFi 5 GHz` или `LAN`:
 - Telegram работает через VPN-туннель **AmneziaWG**
 - YouTube проходит через **youtubeUnblock** и идет провайдеру
 - Остальной трафик идет провайдеру

При подключении по `WiFi 2.4 GHz`:
 - Весь трафик идет через VPN-туннель **AmneziaWG**

***

## Шаг 1: Первоначальная настройка роутера

Включите роутер, подключите кабель провайдера в порт `WAN`, а кабель от ПК в любой порт `LAN`.
На ПК зайдите в Web-интерфейс роутера: [http://192.168.1.1](http://192.168.1.1) (**Имя пользователя**: `root`, **Пароль**: *без пароля*).
Выполните сброс настроек: **Система** -> **Восстановление / Обновление** -> **Выполнить сброс**.
Подождите - откроется **Мастер первоначальной настройки**. Пройдите все шаги, как указано:

1. Интернет
   - **Протокол**: `PPPoE` (*обычно такой - смотрите свой договор на Интернет*)
   - **Имя пользователя**: *ваш логин из договора на Интернет*
   - **Пароль**: *ваш пароль из договора на Интернет*
   - **DNS-серверы**: `77.88.8.8`, `77.88.8.1` (**рекомендую эти, но можно не указывать или сменить на другие**)
   - **Включить IPv6**: `[ ]`
2. Сеть
   - **IP-адрес роутера**: `192.168.1.1`
   - **Маска сети**: `255.255.255.0`
3. WiFi
   - **Одинаковое имя для 2.4 и 5 ГГц**: `[ ]`
   - **Сеть 2.4 ГГц**: `RouteRich_2` (*можете сменить*)
   - **Пароль**: `12345678` (*можете сменить*)
   - **Сеть 5 ГГц**: `RouteRich_5` (*можете сменить*)
   - **Пароль**: `12345678` (*можете сменить*)
4. Mesh
   - **Включить Mesh-сеть**: `[ ]`

## Шаг 2: Настройка туннеля AmneziaWG

Откройте сайт: [WARP Генератор](https://warp-generator.github.io/) и в разделе **AmneziaWG** нажмите кнопку **AWG 2.0 (любой вариант)** и скачайте *файл конфигурации*. Затем в Web-интерфейсе роутера:

1. **Сеть** -> **Интерфейсы** -> **Добавить новый интерфейс..** -> **Имя**: `awg0`, **Протокол**: `AmneziaWG VPN` -> **Создать интерфейс**
2. На вкладке **Основные настройки** -> **Импорт конфигурации**: нажмите **Загрузка конфигурации..** 
3. Откройте блокнотом ранее скачанный *файл конфигурации* (`WARP*.conf`) и вставьте его содержимое в пустое поле.
4. **Импорт настроек** -> **OK** -> **Сохранить** -> **Применить**

## Шаг 3: Настройка интерфейсов, Firewall и маршрутизации

В Web-интерфейсе роутера:

1. **Службы** -> **Терминал** -> **RouteRich login**: `root`
2. Скопируйте команды ниже и вставьте в окно терминала:
```bash
# 1. Устанавливаем пакет Web-интерфейса для youtubeUnblock
# Система -> Пакеты -> Действия: Обновить списки.. -> Закрыть
# Загрузить и установить пакет: luci-app-youtubeUnblock -> OK -> Установить -> Закрыть
for i in $(seq 1 10); do
  opkg list-installed | grep -q luci-app-youtubeUnblock && break
  echo "[ Установка luci-app-youtubeUnblock: попытка $i из 10 ]"
  opkg update -V0 && opkg install luci-app-youtubeUnblock
done

# 2. Добавляем интерфейс awg0 в зону wan
# Сеть -> Межсетевой экран -> Зоны -> wan -> Изменить
# Охватываемые сети: awg0: wan:
# Сохранить -> Применить
uci show firewall | grep "@zone.*name='wan'" | cut -d. -f2 | sort -Vr | while read r; do
  uci -q del_list firewall.$r.network='awg0'
  uci -q add_list firewall.$r.network='awg0'
done

# 3. Создаем бридж br-wifi24
# Сеть -> Интерфейсы -> Устройства -> Добавить конфигурацию устройства..
# На вкладке Общие опции устройства:
#   - Тип устройства: Мост
#   - Имя устройства: br-wifi24
# Сохранить -> Применить
uci set network.br_wifi24='device'
uci set network.br_wifi24.type='bridge'
uci set network.br_wifi24.name='br-wifi24'

# 4. Создаем интерфейс wifi24 и добавляем его в бридж br-wifi24
# Сеть -> Интерфейсы -> Добавить новый интерфейс..
#   - Имя: wifi24
#   - Протокол: Статический адрес
#   - Устройство: br-wifi24
# Создать интерфейс
# На вкладке Основные настройки:
#   - IPv4-адрес: 192.168.2.1
#   - Маска сети IPv4: 255.255.255.0
# На вкладке Расширенные настройки:
#   - Использовать собственные DNS-серверы: 1.1.1.1
#                                           8.8.8.8
uci set network.wifi24='interface'
uci set network.wifi24.proto='static'
uci set network.wifi24.device='br-wifi24'
uci set network.wifi24.ipaddr='192.168.2.1'
uci set network.wifi24.netmask='255.255.255.0'
uci set network.wifi24.dns='1.1.1.1 8.8.8.8'

# 5. Привязываем WiFi 2.4 GHz к интерфейсу wifi24
# Сеть -> Беспроводная сеть -> SSID: RouteRich_2 -> Изменить -> Настройка сети
# На вкладке Основные настройки:
#   - Сеть: wifi24
# Сохранить -> Применить
uci set wireless.default_radio0.network='wifi24'

# 6. Добавляем зону Firewall для интерфейса wifi24
# Сеть -> Межсетевой экран -> Зоны -> Добавить
# На вкладке Основные настройки:
#   - Имя: wifi24
#   - Входящий трафик: Принимать
#   - Выход: Принимать
#   - Внутризональная пересылка: Принимать
#   - Маскарадинг: [x]
#   - Ограничение MSS: [x]
#   - Охватываемые сети: wifi24
# Сохранить -> Применить
# Перед добавлением: Удаление зоны Firewall для интерфейса wifi24
uci show firewall | grep "@zone.*name='wifi24'" | cut -d. -f2 | sort -Vr | while read r; do uci -q delete firewall.$r; done
uci set firewall.wifi24='zone'
uci set firewall.wifi24.name='wifi24'
uci set firewall.wifi24.input='ACCEPT'
uci set firewall.wifi24.output='ACCEPT'
uci set firewall.wifi24.forward='ACCEPT'
uci set firewall.wifi24.masq='1'
uci set firewall.wifi24.mtu_fix='1'
uci set firewall.wifi24.network='wifi24'

# 7. Добавляем правила пересылки между зонами wifi24 и wan
# Сеть -> Межсетевой экран -> Зоны -> wifi24 -> Изменить
# На вкладке Основные настройки:
#   - Разрешить перенаправление в зоны назначения: wan
#   - Разрешить перенаправление из зон источников: wan
# Сохранить -> Применить
# Перед добавлением: Удаление правил пересылки для зоны wifi24
uci show firewall | grep '@forwarding.*wifi24' | cut -d. -f2 | sort -Vr | while read r; do uci -q delete firewall.$r; done
uci add firewall forwarding
uci set firewall.@forwarding[-1].src='wifi24'
uci set firewall.@forwarding[-1].dest='wan'
uci add firewall forwarding
uci set firewall.@forwarding[-1].src='wan'
uci set firewall.@forwarding[-1].dest='wifi24'

# 8. Привязываем интерфейс wifi24 к таблице 100
# Сеть -> Интерфейсы -> wifi24 -> Изменить
# На вкладке Расширенные настройки:
#   - Переопределить таблицу маршрутизации IPv4: 100
# Сохранить -> Применить
uci set network.wifi24.rtable='100'

# 9. Настраиваем DHCP-сервер для интерфейса wifi24
# Сеть -> Интерфейсы -> wifi24 -> Изменить
# На вкладке DHCP-сервер:
#   - [x] Настроить DHCP-сервер:
#     - Запустить: 10
#     - Лимит адресов: 245
# Сохранить -> Применить
uci set dhcp.wifi24='dhcp'
uci set dhcp.wifi24.interface='wifi24'
uci set dhcp.wifi24.start='10'
uci set dhcp.wifi24.limit='245'
uci set dhcp.wifi24.leasetime='12h'

# 10. Добавляем маршрут по умолчанию через интерфейс awg0 в таблице 100
# Сеть -> Маршрутизация
# На вкладке Статические маршруты IPv4:
#   - Добавить:
#     - Интерфейс: awg0
#     - Приоритет: 0.0.0.0/0
#   На вкладке Расширенные настройки:
#     - Таблица: 100
# Сохранить -> Применить
# Перед добавлением: Удаление всех старых маршрутов через интерфейс awg0
uci show network | grep '@route.*awg0' | cut -d. -f2 | sort -Vr | while read r; do uci -q delete network.$r; done
uci add network route
uci set network.@route[-1].interface='awg0'
uci set network.@route[-1].target='0.0.0.0/0'
uci set network.@route[-1].table='100'

# 11. Добавляем маршрут в 192.168.2.0/24 через интерфейс wifi24 в таблице 100
# Сеть -> Маршрутизация
# На вкладке Статические маршруты IPv4:
#   - Добавить:
#     - Интерфейс: wifi24
#     - Приоритет: 192.168.2.0/24
#   На вкладке Расширенные настройки:
#     - Таблица: 100
# Сохранить -> Применить
# Перед добавлением: Удаление всех старых маршрутов через интерфейс wifi24
uci show network | grep '@route.*wifi24' | cut -d. -f2 | sort -Vr | while read r; do uci -q delete network.$r; done
uci add network route
uci set network.@route[-1].interface='wifi24'
uci set network.@route[-1].target='192.168.2.0/24'
uci set network.@route[-1].table='100'

# 12. Добавляем маршруты к Telegram через интерфейс awg0 в таблице main
# Сеть -> Маршрутизация -> Статические маршруты IPv4
# На вкладке Статические маршруты IPv4:
#   - Добавить
#     - Интерфейс: awg0
#     - Приоритет: 91.105.192.0/23
# Сохранить -> Применить (повторяем этот шаг для каждой подсети из telegram_nets)
# Перед добавлением: Удаление всех старых маршрутов через интерфейс awg0 в таблице main
uci show network | grep '@route.*awg0' | cut -d. -f2 | sort -Vr | while read r; do [ "$(uci -q get network.$r.table)" ] || uci -q delete network.$r; done
telegram_nets='91.105.192.0/23 91.108.4.0/22 91.108.8.0/21 91.108.16.0/21 91.108.56.0/22 95.161.64.0/20 149.154.160.0/20 185.76.151.0/24'
for net in $telegram_nets; do
  uci add network route
  uci set network.@route[-1].interface='awg0'
  uci set network.@route[-1].target="$net"
done

# 13. Создаем правило маршрутизации: трафик из 192.168.2.0/24 -> таблица 100
# На вкладке Правила IPv4:
#   - Добавить:
#   На вкладке Основные настройки:
#     - Источник: 192.168.2.0/24
#   На вкладке Расширенные настройки:
#     - Таблица: 100
# Сохранить -> Применить
# Перед добавлением: Удаление всех старых правил для сети 192.168.2.0/24
uci show network | grep "@rule.*src='192.168.2.0/24'" | cut -d. -f2 | sort -Vr | while read r; do uci -q delete network.$r; done
uci add network rule
uci set network.@rule[-1].src='192.168.2.0/24'
uci set network.@rule[-1].lookup='100'

# 14. Добавляем скрипт автозапуска для youtubeUnblock в rc.local
# Система -> Автозапуск -> Запуск пакетов и служб пользователя, при включении устройства
cat > /etc/rc.local << 'EOF'
# Исключаем AWG-трафик из обработки youtubeUnblock
for i in $(seq 1 60); do
  if ip link show awg0 && nft list chain inet fw4 youtubeUnblock; then
    sleep 5
    nft insert rule inet fw4 youtubeUnblock oifname 'awg0' counter return comment 'skip_awg'
    break
  fi
  sleep 2
done
exit 0
EOF
chmod +x /etc/rc.local

# 15. Заставляем службу Терминал слушать на всех интерфейсах вместо lan
# Службы -> Терминал -> Конфигурация -> Интерфейс: не определено
uci -q delete ttyd.@ttyd[0].interface

# 16. Сохраняем все изменения
# Сеть -> Интерфейсы -> Применить
# Сеть -> Маршрутизация -> Применить
# Сеть -> Межсетевой экран -> Применить
# Сеть -> Беспроводная сеть -> Применить
# Сеть -> DHCP и DNS -> Применить
# Службы -> Терминал -> Конфигурация -> Применить
uci commit network
uci commit firewall
uci commit wireless
uci commit dhcp
uci commit ttyd

# 17. Перезагружаемся
# Система -> Перезагрузка -> Выполнить перезагрузку
reboot
```
3. После выполнения команд в терминале нажмите **Enter** для перезагрузки.

***

## Тестирование после перезагрузки


В Web-интерфейсе роутера:

1. **Службы** -> **Терминал** -> **RouteRich login**: `root`
2. Скопируйте команды ниже и вставьте в окно терминала:
```bash
sh << 'SCRIPT'
check(){ r="31m[-]"; [ "$(eval "$2" 2>/dev/null)" ] && r="32m[+]"; printf "\033[1;%s\033[0m %s\n" "$r" "$1"; }
check "Шаг 1/1: IPv6 отключен"                                                            "uci show | grep \"wizard.default.ipv6='0'\""
check "Шаг 1/2: IP-адрес роутера: 192.168.1.1"                                            "uci get network.lan.ipaddr | grep '192.168.1.1'"
check "Шаг 1/2: Маска сети: 255.255.255.0"                                                "uci get network.lan.netmask | grep '255.255.255.0'"
check "Шаг 1/3: Разные SSID для 2.4 и 5 ГГц"                                              "uci get wizard.default.unify_ssid | grep 0"
check "Шаг 1/4: Mesh-сеть отключена"                                                      "uci show | grep \"mesh_enabled='0'\""
check "Шаг 2/1: Интерфейс awg0 создан с протоколом AmneziaWG"                             "uci get network.awg0.proto | grep amneziawg"
check "Шаг 2/2: Приватный ключ настроен"                                                  "uci get network.awg0.private_key | grep ."
check "Шаг 2/2: Endpoint сервера настроен"                                                "uci show network | grep 'awg0.*endpoint_host'"
check "Шаг 2/2: Туннель активен"                                                          "awg show awg0 | grep handshake | grep -v never"
check "Шаг 3/1: Устанавливаем пакет Web-интерфейса для youtubeUnblock"                    "opkg list-installed | grep luci-app-youtubeUnblock"
check "Шаг 3/2: Добавляем интерфейс awg0 в зону wan"                                      "uci show firewall | grep awg0"
check "Шаг 3/3: Создаем бридж br-wifi24"                                                  "ip addr show br-wifi24 | grep 'inet 192.168.2.1/24 brd 192.168.2.255'"
check "Шаг 3/4: Создаем интерфейс wifi24 и добавляем его в бридж br-wifi24"               "uci get network.wifi24.device | grep br-wifi24"
check "Шаг 3/5: Привязываем WiFi 2.4 GHz к интерфейсу wifi24"                             "uci get wireless.default_radio0.network | grep wifi24"
check "Шаг 3/6: Добавляем зону Firewall для интерфейса wifi24"                            "uci get firewall.wifi24.network | grep wifi24"
check "Шаг 3/7: Добавляем правила пересылки между зонами wifi24 и wan"                    "uci show firewall | grep '@forwarding' | grep wifi24"
check "Шаг 3/8: Привязываем интерфейс wifi24 к таблице 100"                               "uci get network.wifi24.rtable | grep 100"
check "Шаг 3/9: Настраиваем DHCP-сервер для интерфейса wifi24"                            "uci get dhcp.wifi24.interface | grep wifi24"
check "Шаг 3/10: Добавляем маршрут по умолчанию через интерфейс awg0 в таблице 100"       "ip route show table 100 | grep 'default dev awg0'"
check "Шаг 3/11: Добавляем маршрут в 192.168.2.0/24 через интерфейс wifi24 в таблице 100" "ip route show table 100 | grep '192.168.2.0/24 dev br-wifi24'"
check "Шаг 3/12: Добавляем маршруты к Telegram через awg0 в таблице main"                 "ip route show table main | grep awg0 | grep '91.105.192.0/23'"
check "Шаг 3/13: Создаем правило маршрутизации: 192.168.2.0/24 -> таблица 100"            "ip rule show | grep 'from 192.168.2.0/24 lookup 100'"
check "Шаг 3/14: Добавляем скрипт автозапуска для youtubeUnblock в rc.local"              "nft list chain inet fw4 youtubeUnblock | grep skip_awg"
check "Шаг 3/15: Заставляем службу Терминал слушать на всех интерфейсах вместо lan"       "netstat -tlnp | grep ttyd | grep '0.0.0.0:7681'"
SCRIPT
```

Если вывод показывает, что все шаги успешно пройдены, можете проверять работу роутера:

1. Подключитесь к `WiFi 2.4 GHz` (**Routerich_2**)
   - Проверьте IP-адрес - должен быть: `192.168.2.x`
   - Откройте [ifconfig.me](https://ifconfig.me) - должен быть IP-адрес сервера **AmneziaWG**
   - Проверьте работу **[YouTube](https://www.youtube.com/)** и других заблокированных ресурсов, например **[X](https://x.com/)**
2. Подключитесь к `WiFi 5 GHz` (**Routerich_5**) или `LAN`
   - Проверьте IP-адрес - должен быть: `192.168.1.x`
   - Откройте [ifconfig.me](https://ifconfig.me) - должен быть IP-адрес вашего провайдера
   - Проверьте работу **Telegram** и **[YouTube](https://www.youtube.com/)**
