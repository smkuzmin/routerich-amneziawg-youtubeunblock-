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

Включите роутер, подключите кабель провайдера в порт `WAN`, а кабель от ПК в любой порт `LAN`. Затем откройте [Мастер настроек Routerich](http://192.168.1.1/luci-static/resources/wizard.html) (**Имя пользователя**: `root`, **Пароль**: *без пароля*) и пройдите все шаги мастера, заполняя соответствующие поля:

1. Интернет
   - **Протокол**: `PPPoE` (*обычно такой - смотрите свой договор на Интернет*)
   - **Имя пользователя**: *ваш логин из договора на Интернет*
   - **Пароль**: *ваш пароль из договора на Интернет*
   - **DNS-серверы**: `77.88.8.8`, `77.88.8.1`
   - **Включить IPv6**: `[ ]`
2. Сеть
   - **IP-адрес роутера**: `192.168.1.1`
   - **Маска сети**: `255.255.255.0`
3. WiFi
   - **Одинаковое имя для 2.4 и 5 ГГц**: `[ ]`
   - **Сеть 2.4 ГГц**: `RouteRich_2`
   - **Пароль**: `12345678`
   - **Сеть 5 ГГц**: `RouteRich_5`
   - **Пароль**: `12345678`
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
opkg update
opkg install luci-app-youtubeUnblock

# 2. Добавляем интерфейс awg0 в зону wan
# Сеть -> Межсетевой экран -> Зоны -> wan -> Изменить
# Охватываемые сети: добавить awg0 -> Сохранить -> Применить
for z in $(uci show firewall | grep '@zone' | grep "name='wan'" | cut -d. -f2 | cut -d= -f1); do
  uci -q del_list firewall.$z.network='awg0'
  uci add_list firewall.$z.network='awg0'
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

# 4. Создаем интерфейс wifi24 и добавляем его в бридж
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
# Сеть -> Беспроводная сеть -> SSID: RouteRich_2 -> Изменить -> Настройка сети -> Сеть: заменяем lan на wifi24 -> Сохранить -> Применить
uci set wireless.default_radio0.network='wifi24'

# 6. Создаем зону Firewall для интерфейса wifi24
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
for z in $(uci show firewall | grep '@zone' | grep "name='wifi24'" | cut -d. -f2 | cut -d= -f1); do
  uci delete firewall.$z
done
uci set firewall.wifi24='zone'
uci set firewall.wifi24.name='wifi24'
uci set firewall.wifi24.input='ACCEPT'
uci set firewall.wifi24.output='ACCEPT'
uci set firewall.wifi24.forward='ACCEPT'
uci set firewall.wifi24.masq='1'
uci set firewall.wifi24.mtu_fix='1'
uci set firewall.wifi24.network='wifi24'

# 7. Разрешаем forward между зонами
# Сеть -> Межсетевой экран -> Зоны -> wifi24 -> Изменить
# На вкладке Основные настройки:
#   - Разрешить перенаправление в зоны назначения: wan
#   - Разрешить перенаправление из зон источников: wan
# Сохранить -> Применить
uci show firewall | grep "@forwarding" | grep "src='wifi24'\|dest='wifi24'" | cut -d. -f2 | while read r; do
  uci delete firewall.$r >/dev/null 2>&1
done
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
# Сохранить -> Применить
uci set dhcp.wifi24='dhcp'
uci set dhcp.wifi24.interface='wifi24'
uci set dhcp.wifi24.start='100'
uci set dhcp.wifi24.limit='150'
uci set dhcp.wifi24.leasetime='12h'

# 10. Удаляем все старые маршруты через AWG и правила для 192.168.2.0/24
# Сеть -> Маршрутизация -> удаляем вручную соответствующие записи
uci show network | grep "@route" | grep "awg0"           | cut -d. -f2 | while read r; do uci delete network.$r >/dev/null 2>&1; done
uci show network | grep "@rule"  | grep "192.168.2.0/24" | cut -d. -f2 | while read r; do uci delete network.$r >/dev/null 2>&1; done

# 11. Добавляем маршрут по умолчанию через интерфейс awg0 в таблице 100
# Сеть -> Маршрутизация
# На вкладке Статические маршруты IPv4:
#   - Добавить:
#     - Интерфейс: awg0
#     - Приоритет: 0.0.0.0/0
#   На вкладке Расширенные настройки:
#     - Таблица: 100
# Сохранить -> Применить
uci add network route
uci set network.@route[-1].interface='awg0'
uci set network.@route[-1].target="0.0.0.0/0"
uci set network.@route[-1].table='100'

# 12. Добавляем локальный маршрут через интерфейс wifi24 в таблице 100
# Сеть -> Маршрутизация
# На вкладке Статические маршруты IPv4:
#   - Добавить:
#     - Интерфейс: wifi24
#     - Приоритет: 192.168.2.0/24
#   На вкладке Расширенные настройки:
#     - Таблица: 100
# Сохранить -> Применить
uci add network route
uci set network.@route[-1].interface='wifi24'
uci set network.@route[-1].target="192.168.2.0/24"
uci set network.@route[-1].table='100'

# 13. Добавляем маршруты Telegram через интерфейс awg0 в основной таблице
# Сеть -> Маршрутизация -> Статические маршруты IPv4
# На вкладке Статические маршруты IPv4:
#   - Добавить
#     - Интерфейс: awg0
#     - Приоритет: 91.105.192.0/23
# Сохранить -> Применить (повторяем для каждой подсети)
for net in 91.105.192.0/23 91.108.4.0/22 91.108.8.0/21 91.108.16.0/21 91.108.56.0/22 95.161.64.0/20 149.154.160.0/20 185.76.151.0/24; do
  uci add network route
  uci set network.@route[-1].interface='awg0'
  uci set network.@route[-1].target="$net"
done

# 14. Создаем правило маршрутизации: трафик из 192.168.2.0/24 -> таблица 100
# На вкладке Правила IPv4:
#   - Добавить:
#   На вкладке Основные настройки:
#     - Источник: 192.168.2.0/24
#   На вкладке Расширенные настройки:
#     - Таблица: 100
# Сохранить -> Применить
uci add network rule
uci set network.@rule[-1].src='192.168.2.0/24'
uci set network.@rule[-1].lookup='100'

# 15. Сохраняем все изменения
# Сеть -> Интерфейсы -> Применить
# Сеть -> Маршрутизация -> Применить
# Сеть -> Межсетевой экран -> Применить
# Сеть -> Беспроводная сеть -> Применить
# Сеть -> DHCP и DNS -> Применить
uci commit network
uci commit firewall
uci commit wireless
uci commit dhcp

# 16. Включаем youtubeUnblock для автозапуска
# Система -> Автозапуск -> youtubeUnblock -> Включено
/etc/init.d/youtubeUnblock enable

# 17. Перезапускаем Firewall, чтобы создать цепочку youtubeUnblock
/etc/init.d/firewall restart

# 18. Добавляем скрипт автозапуска для youtubeUnblock в rc.local
# Система -> Автозапуск -> Запуск пакетов и служб пользователя, при включении устройства
cat > /etc/rc.local << 'EOF'
# Исключаем AWG-трафик из обработки youtubeUnblock
for i in $(seq 1 60); do
  if ip link show awg0 >/dev/null 2>&1 && nft list chain inet fw4 youtubeUnblock >/dev/null 2>&1; then
    sleep 5
    nft insert rule inet fw4 youtubeUnblock oifname "awg0" counter return comment "skip_awg" 2>/dev/null
    break
  fi
  sleep 2
done

exit 0
EOF
chmod +x /etc/rc.local

# 19. Перезагружаемся
# Система -> Перезагрузка -> Выполнить перезагрузку
reboot
```

***

## Тестирование после перезагрузки

1. Подключитесь к `WiFi 2.4 GHz` (**Routerich_2**)
2. Проверьте IP-адрес - должен быть: `192.168.2.x`
3. Откройте [ifconfig.me](https://ifconfig.me) - должен быть IP AWG-сервера
4. Проверьте работу **[YouTube](https://www.youtube.com/)** и других заблокированных ресурсов, например **[X](https://x.com/)**
5. Подключитесь к `WiFi 5 GHz` (**Routerich_5**) или `LAN`
6. Проверьте IP-адрес - должен быть: `192.168.1.x`
7. Откройте [ifconfig.me](https://ifconfig.me) - должен быть IP вашего провайдера
8. Проверьте работу **[YouTube](https://www.youtube.com/)**

***

### Скрипт для быстрой диагностики

В Web-интерфейсе роутера:

1. **Службы** -> **Терминал** -> **RouteRich login**: `root`
2. Скопируйте команды ниже и вставьте в окно терминала:
```bash
sh << 'SCRIPT'
check(){ local out=$(eval "$2" 2>/dev/null); local r="-"; [ "$out" ] && r="+"; printf "[%s] %s\n" "$r" "$1"; }
check "WiFi 2.4 привязан к интерфейсу wifi24" "uci get wireless.default_radio0.network"
check "Интерфейс br-wifi24 поднят" "ip addr show br-wifi24 2>/dev/null | grep 'inet '"
check "DHCP раздаёт адреса в 192.168.2.x" "awk '\$3 ~ /^192\\.168\\.2\\./ {print \$3, \$4}' /tmp/dhcp.leases"
check "Правило маршрутизации существует" "ip rule show | grep '192.168.2.0/24'"
check "В таблице 100 есть маршрут по умолчанию через awg0" "ip route show table 100 | grep 'default dev awg0'"
check "В таблице 100 есть локальный маршрут" "ip route show table 100 | grep '192.168.2.0/24 dev br-wifi24'"
check "Firewall-зона wifi24 существует" "uci get firewall.wifi24.network"
check "AWG-туннель активен" "awg show | grep 'latest handshake'"
check "youtubeUnblock запущен" "/etc/init.d/youtubeUnblock status | grep 'running'"
check "Правило skip_awg есть в цепочке" "nft list chain inet fw4 youtubeUnblock | grep 'skip_awg'"
check "WiFi 2.4 GHz работает" "iwinfo | grep 'RouteRich_2'"
SCRIPT
```
