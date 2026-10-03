# КР АДСЕТИ — по подпунктам, простыми командами (домен au-team.irpo, пароль везде P@ssw0rd)

Иди сверху вниз: подпункт N+1 делай, когда N готов. В каждом подпункте только своё: **где** → **команды** → **проверка**.
Если проверка не сошлась, дальше не иди.

## Сначала: будь root на Linux-машинах

Все команды для ISP, HQ-SW, HQ-SRV, HQ-CLI, BR-SRV вводятся от root. Приглашение `#` — ты root, `$` — обычный пользователь (команды дадут «Permission denied» или «only superuser…»).
Стать root:
```
su -
```
Пароль root — из данных стенда (в лабах `toor`). Не получилось — `sudo -i` с паролем своего пользователя.
Проверка: `whoami` должен вывести `root`.

## Что нужно запомнить (хватит на всю КР)

**Linux, сеть:**
- `nmcli con show` — список подключений (смотри имя)
- `nmcli con del "имя"` — удалить подключение
- `nmcli con add con-name lan ifname ens19 type ethernet ip4 АДРЕС/МАСКА gw4 ШЛЮЗ` — создать
- `nmcli con mod lan ipv4.dns АДРЕС` — прописать DNS
- `systemctl restart NetworkManager` — применить
- `ip -br a` — посмотреть адреса · `ping -c2 АДРЕС` — проверить связь

**Linux, службы:** `systemctl enable --now ИМЯ` (включить и запустить) · `systemctl restart ИМЯ` · `systemctl is-active ИМЯ`

**Редактор nano:** `nano /путь/к/файлу` — открыть.
Стрелки — двигаться · `Ctrl+W` — поиск · `Ctrl+O`, `Enter` — сохранить · `Ctrl+X` — выйти.
Строки в конец файла: `Ctrl+End`, потом печатай/вставляй (вставка — правая кнопка мыши).
Нет nano → `apt-get install nano` (после 8.2) или открой тем же путём через `vi`.

**ESR (роутеры):** `configure` → команды → `end` → `commit` → `confirm`. Без `confirm` через 10 минут всё откатится.

## Карта: что на каком порту

| Устройство | Порты |
|---|---|
| ISP | ens19 интернет · ens20 → HQ-RTR · ens21 → BR-RTR |
| HQ-RTR | gi1/0/2 → ISP · gi1/0/1 → HQ-SW |
| BR-RTR | gi1/0/1 → ISP · gi1/0/2 → BR-SRV |
| HQ-SW | ens19 → HQ-RTR · ens20 → HQ-SRV · ens21 → HQ-CLI |
| HQ-SRV, HQ-CLI, BR-SRV | ens19 |

⚠ Порты везде как в схеме КР. Если на стенде связи нет при верных командах — сверь порты по MAC: на ESR `show interfaces status`, в virt-manager у каждой карты видна её сеть. Совпал MAC с другой сетью — просто поменяй порты местами в командах (на роутере `gi1/0/1` ↔ `gi1/0/2`). Имена портов Linux смотри `ip -br link`: в файле `ens19/20/21` — по схеме, у тебя могут быть `enp1s0/2/3` — подставляй свои в тех же ролях (к роутеру / к SRV / к CLI).

## Таблица адресов (всё уже посчитано)

| Что | Сеть | Роутер (шлюз) | Хост |
|---|---|---|---|
| ISP–HQ-RTR | 172.16.1.0/28 | ISP .1 | HQ-RTR .2 |
| ISP–BR-RTR | 172.16.2.0/28 | ISP .1 | BR-RTR .2 |
| HQ-SRV, VLAN 150 | 192.100.100.0/25 | HQ-RTR .1 | HQ-SRV .2 |
| HQ-CLI, VLAN 250 | 192.100.200.0/26 | HQ-RTR .1 | HQ-CLI .2 |
| Управление, VLAN 333 | 192.100.30.0/29 | HQ-RTR .1 | HQ-SW .2 |
| BR-SRV | 192.100.20.0/28 | BR-RTR .1 | BR-SRV .2 |
| Туннель | 10.10.10.0/30 | HQ-RTR .1 | BR-RTR .2 |

---

# 1. Базовая настройка устройств

### 1.1 Имена устройств с полным доменным именем

🖥 Linux (ISP, HQ-SW, HQ-SRV, HQ-CLI, BR-SRV) — на каждой своё имя:
```
hostnamectl set-hostname isp.au-team.irpo
exec bash
```
Имена: `isp.au-team.irpo` · `hq-sw.au-team.irpo` · `hq-srv.au-team.irpo` · `hq-cli.au-team.irpo` · `br-srv.au-team.irpo`

🖥 HQ-RTR (логин admin / пароль password):
```
configure
hostname hq-rtr.au-team.irpo
end
commit
confirm
```
🖥 BR-RTR: то же, но `hostname br-rtr.au-team.irpo`.

🔍 Linux: `hostname` — полное имя. ESR: имя в приглашении.
🛠 ESR не принял точку → `hostname hq-rtr` и `domain name au-team.irpo`.

### 1.2 IPv4 на всех устройствах

Общий пункт, подпункты 1.2.1–1.2.5 ниже.

### 1.2.1 IP из приватного диапазона

Приватные здесь внешние сети 172.16.1.0/28 и 172.16.2.0/28. Ставим адреса роутерам на сторону ISP. Сети 192.100.x заданы условием, их менять нельзя.

🖥 HQ-RTR:
```
configure
interface gigabitethernet 1/0/2
ip firewall disable
ip address 172.16.1.2/28
end
commit
confirm
```
🖥 BR-RTR:
```
configure
interface gigabitethernet 1/0/1
ip firewall disable
ip address 172.16.2.2/28
end
commit
confirm
```
🔍 `show ip interfaces` — адрес на порту, `Up`.
🛠 порт не поднимается или есть bridge/security-zone → внутри interface: `no bridge-group`, `no security-zone`, `no ip address dhcp`.

### 1.2.2 Сеть HQ-SRV (VLAN 150): ≤128 адресов из 192.100.100.0/24 → 192.100.100.0/25

🖥 HQ-RTR (родительский порт и сабинтерфейс VLAN 150):
```
configure
interface gigabitethernet 1/0/1
ip firewall disable
exit
interface gigabitethernet 1/0/1.150
ip firewall disable
ip address 192.100.100.1/25
end
commit
confirm
```
🖥 HQ-SRV:
```
nmcli con show
nmcli con del "Проводное подключение 1"
nmcli con add con-name lan ifname ens19 type ethernet ip4 192.100.100.2/25 gw4 192.100.100.1
nmcli con mod lan ipv4.dns 94.232.137.104
systemctl restart NetworkManager
```
(Имя в `nmcli con del` бери из вывода `nmcli con show`, столбец NAME у порта ens19.)

🔍 HQ-RTR: `show ip interfaces` — `gi1/0/1.150` `192.100.100.1/25`. HQ-SRV: `ip -br a` — `192.100.100.2/25`. Пинг до шлюза заработает после 4.1–4.3.

### 1.2.3 Сеть HQ-CLI (VLAN 250): ≤64 адресов из 192.100.200.0/24 → 192.100.200.0/26

🖥 HQ-RTR:
```
configure
interface gigabitethernet 1/0/1.250
ip firewall disable
ip address 192.100.200.1/26
end
commit
confirm
```
🖥 HQ-CLI:
```
nmcli con show
nmcli con del "Проводное подключение 1"
nmcli con add con-name lan ifname ens19 type ethernet ip4 192.100.200.2/26 gw4 192.100.200.1
nmcli con mod lan ipv4.dns 94.232.137.104
systemctl restart NetworkManager
```
🔍 HQ-RTR: `gi1/0/1.250` `192.100.200.1/26`. HQ-CLI: `ip -br a` — `192.100.200.2/26`.

### 1.2.4 Сеть управления (VLAN 333): ≤8 адресов из 192.100.30.0/28 → 192.100.30.0/29

🖥 HQ-RTR:
```
configure
interface gigabitethernet 1/0/1.333
ip firewall disable
ip address 192.100.30.1/29
end
commit
confirm
```
Адрес HQ-SW (192.100.30.2/29) задаётся в подпункте 4.3.
🔍 HQ-RTR: `gi1/0/1.333` `192.100.30.1/29`.

### 1.2.5 Сеть BR-SRV: ≤16 адресов из 192.100.20.0/26 → 192.100.20.0/28

🖥 BR-RTR:
```
configure
interface gigabitethernet 1/0/2
ip firewall disable
ip address 192.100.20.1/28
end
commit
confirm
```
🖥 BR-SRV:
```
nmcli con show
nmcli con del "Проводное подключение 1"
nmcli con add con-name lan ifname ens19 type ethernet ip4 192.100.20.2/28 gw4 192.100.20.1
nmcli con mod lan ipv4.dns 94.232.137.104
systemctl restart NetworkManager
```
🔍 BR-SRV: `ip -br a` — `192.100.20.2/28` · `ping -c2 192.100.20.1` — 0% loss.

---

# 2. Доступ в интернет, настройка ISP

### 2.1 Адресация на интерфейсах ISP

**2.1.1 Интерфейс к провайдеру — адрес по DHCP.** Стенд обычно уже настроил его.

🖥 ISP:
```
ip -br a
```
🔍 на `ens19` есть адрес. Нет — создай DHCP-подключение:
```
nmcli con add con-name inet ifname ens19 type ethernet
nmcli con up inet
```
⚠ Подключение на ens19 НЕ удаляй.

**2.1.2 Маршрут по умолчанию (если необходимо).** Приходит по DHCP, руками обычно не нужен.

🖥 ISP:
```
ip route
```
🔍 есть строка `default via …`. Нет — `nmcli con up inet`.

**2.1.3 Интерфейс в сторону HQ-RTR, сеть 172.16.1.0/28.**

🖥 ISP:
```
nmcli con show
nmcli con del "Проводное подключение 2"
nmcli con add con-name toHQ ifname ens20 type ethernet ip4 172.16.1.1/28
```
(Удаляй то подключение, у которого в столбце DEVICE стоит ens20.)
🔍 `ip -br a` — `ens20 172.16.1.1/28` · `ping -c2 172.16.1.2`.

**2.1.4 Интерфейс в сторону BR-RTR, сеть 172.16.2.0/28.**

🖥 ISP:
```
nmcli con del "Проводное подключение 3"
nmcli con add con-name toBR ifname ens21 type ethernet ip4 172.16.2.1/28
```
(Опять же — подключение с DEVICE ens21.)
🔍 `ip -br a` — `ens21 172.16.2.1/28` · `ping -c2 172.16.2.2`.

### 2.2 На ISP динамическая трансляция портов (NAT) для HQ-RTR и BR-RTR

🖥 ISP.

Шаг 1 — включить пересылку пакетов:
```
nano /etc/net/sysctl.conf
```
Найди (`Ctrl+W`, слово `forward`) строку `net.ipv4.ip_forward = 0`, замени `0` на `1`. Сохрани: `Ctrl+O`, `Enter`, `Ctrl+X`. Затем:
```
sysctl -w net.ipv4.ip_forward=1
```
Шаг 2 — NAT:
```
iptables -t nat -A POSTROUTING -o ens19 -j MASQUERADE
iptables-save > /etc/sysconfig/iptables
systemctl enable --now iptables
```
🔍 `sysctl net.ipv4.ip_forward` — `= 1` · `iptables -t nat -S` — `-A POSTROUTING -o ens19 -j MASQUERADE` · `ping -c2 8.8.8.8` — 0% loss.
🛠 в sysctl.conf нет строки ip_forward → допиши в конец `net.ipv4.ip_forward = 1` · iptables не найден → `apt-get install iptables`.

---

# 3. Локальные учётные записи

### 3.1 На серверах HQ-SRV и BR-SRV: sshuser, пароль P@ssw0rd, UID 2525, sudo без пароля

🖥 HQ-SRV и BR-SRV (на обоих), **от root** (`whoami` → `root`, иначе `su -`).
Если раньше `useradd` уже вводился не от root, проверь: `id sshuser`. Пользователь есть с UID 2525 — пропусти `useradd` и сразу задай пароль. Нет — выполни `useradd`.

3.1.1 + 3.1.3 — пользователь с UID 2525:
```
useradd -m -u 2525 sshuser
```
3.1.2 — задай пароль (вводится дважды, на экране не виден). Команда:
```
passwd sshuser
```
3.1.4 — sudo без пароля:
```
usermod -aG wheel sshuser
control sudo public
nano /etc/sudoers
```
В самый конец файла добавь строку:
```
sshuser ALL=(ALL:ALL) NOPASSWD: ALL
```
Сохрани (`Ctrl+O`, `Enter`, `Ctrl+X`) и проверь файл:
```
visudo -c
```
🔍 `id sshuser` — `uid=2525(sshuser)` · `su - sshuser`, потом `sudo -n id` — `uid=0(root)` без запроса пароля, `sudo -l` — `NOPASSWD: ALL` (выйти: `exit`) · `visudo -c` — `успешно обработан`; предупреждения про `WHEEL_USERS` и про права `99-sudopw` — норма.
🛠 `control: sudo не найден` → вернись после 8.2 (`apt-get install sudo`) · `visudo -c` ругается на файл в /etc/sudoers.d (bad permissions), но в конце `parsed OK` — норма.

### 3.2 На маршрутизаторах HQ-RTR и BR-RTR: admineco, пароль P@ssw0rd

🖥 HQ-RTR и BR-RTR (на обоих):
```
configure
username admineco
password P@ssw0rd
privilege 15
end
commit
confirm
```
🔍 `show running-config` — блок `username admineco`, `privilege 15` · `logout`, вход `admineco` / `P@ssw0rd` — успешно.

---

# 4. Коммутация в сегменте HQ

### 4.1 Трафик HQ-SRV → VLAN 150

🖥 HQ-SW. Сначала посмотри и удали профили на портах ens19, ens20, ens21:
```
nmcli con show
nmcli con del "Проводное подключение 1"
nmcli con del "Проводное подключение 2"
nmcli con del "Проводное подключение 3"
```
(Удаляй те, у которых DEVICE ens19, ens20, ens21; имена бери из списка.)

Создай мост с фильтрацией VLAN и порт к серверу:
```
nmcli con add type bridge con-name br0 ifname br0 bridge.stp no bridge.vlan-filtering yes bridge.vlans 333 ipv4.method disabled ipv6.method disabled
nmcli con add type bridge-slave con-name srv ifname ens20 master br0 bridge-port.vlans "150 pvid untagged"
```
🔍 `bridge vlan show` — у `ens20` `150 PVID Egress Untagged`.

### 4.2 Трафик HQ-CLI → VLAN 250

🖥 HQ-SW:
```
nmcli con add type bridge-slave con-name cli ifname ens21 master br0 bridge-port.vlans "250 pvid untagged"
```
🔍 `bridge vlan show` — у `ens21` `250 PVID Egress Untagged`.

### 4.3 Передача трафика управления в VLAN 333 (trunk к роутеру + адрес HQ-SW)

🖥 HQ-SW:
```
nmcli con add type bridge-slave con-name up ifname ens19 master br0 bridge-port.vlans "150,250,333"
nmcli con add type vlan con-name mgmt ifname mgmt dev br0 id 333 ip4 192.100.30.2/29 gw4 192.100.30.1
nmcli con mod mgmt ipv4.dns 94.232.137.104
systemctl restart NetworkManager
nmcli con up mgmt
```
🔍 `bridge vlan show` — у `ens19` VLAN 150, 250, 333 · `ping -c2 192.100.30.1` — 0% loss.
🛠 у mgmt нет адреса → `nmcli con up br0`, потом `nmcli con up mgmt` · шлюз не пингуется → на HQ-RTR `show ip interfaces`: есть ли `gi1/0/1.333` · мост создан без `bridge.stp no` → `nmcli con mod br0 bridge.stp no`, `systemctl restart NetworkManager`, подожди 10–30 секунд (проверка: `cat /sys/class/net/br0/bridge/stp_state` → `0`).

### 4.4 Маршрутизация всех VLAN на HQ-RTR через один физический порт

Уже сделано в 1.2.2–1.2.4: порт `gi1/0/1` один, на нём три сабинтерфейса `.150`, `.250`, `.333` (router-on-a-stick).
🔍 HQ-RTR: `show ip interfaces` — три сабинтерфейса `Up` · с HQ-SRV `ping -c2 192.100.100.1` — 0% loss · с HQ-CLI `ping -c2 192.100.200.1` — 0% loss · с HQ-SRV `ping -c2 192.100.200.2` — 0% loss (между VLAN идёт через роутер).
🛠 SRV или CLI не видят шлюз → проверь, что кабели ens20 и ens21 не перепутаны, и `bridge vlan show`.

### 4.5 Сведения о коммутации — в отчёт

Текст для отчёта: «На HQ-SW создан мост br0 с фильтрацией VLAN. Порт ens19 к HQ-RTR — trunk (VLAN 150, 250, 333), ens20 к HQ-SRV — access VLAN 150, ens21 к HQ-CLI — access VLAN 250. Управление — интерфейс mgmt в VLAN 333 (192.100.30.2/29). На HQ-RTR на физическом порту gi1/0/1 созданы сабинтерфейсы .150, .250, .333 — маршрутизация между VLAN выполняется роутером (router-on-a-stick)».

---

# 5. Безопасный удалённый доступ на HQ-SRV и BR-SRV

### 5.1 Ограничить число попыток входа до двух

🖥 HQ-SRV и BR-SRV (на обоих):
```
nano /etc/openssh/sshd_config
```
Найди (`Ctrl+W`, слово `MaxAuthTries`) строку `#MaxAuthTries 6` и сделай:
```
MaxAuthTries 2
```
(убери `#`, замени 6 на 2). Найди `PasswordAuthentication` и убедись, что строка `PasswordAuthentication yes` без `#`. Сохрани и перезапусти службу:
```
systemctl restart sshd
```
🔍 `cat /etc/openssh/sshd_config` — видны `MaxAuthTries 2` и `PasswordAuthentication yes` без `#`.

### 5.2 Баннер «Only authorized access is allowed»

🖥 HQ-SRV и BR-SRV (на обоих):
```
nano /etc/openssh/banner
```
Впиши одну строку:
```
Only authorized access is allowed
```
Сохрани. Потом:
```
nano /etc/openssh/sshd_config
```
Найди (`Ctrl+W`, слово `Banner`) строку `#Banner none` и сделай:
```
Banner /etc/openssh/banner
```
Сохрани и примени:
```
sshd -t
systemctl enable --now sshd
systemctl restart sshd
```
🔍 `systemctl is-active sshd` — `active`. С HQ-CLI `ssh sshuser@192.100.100.2` — показан баннер, затем запрос пароля; после двух неверных паролей — `Too many authentication failures`. Для BR-SRV (`192.100.20.2`) проверка возможна после 7.

---

# 6. Туннель между офисами HQ и BR (GRE)

### 6.1 Технология GRE · 6.2 сеть 10.10.10.0/24 · 6.3 минимально возможная маска

Для двух устройств минимальная маска /30 → подсеть 10.10.10.0/30: HQ-RTR .1, BR-RTR .2.

🖥 HQ-RTR:
```
configure
tunnel gre 1
ttl 64
ip firewall disable
local address 172.16.1.2
remote address 172.16.2.2
ip address 10.10.10.1/30
enable
end
commit
confirm
```
🖥 BR-RTR:
```
configure
tunnel gre 1
ttl 64
ip firewall disable
local address 172.16.2.2
remote address 172.16.1.2
ip address 10.10.10.2/30
enable
end
commit
confirm
```
🔍 HQ-RTR: `ping 172.16.2.2` — `!!!!!` (внешние адреса видят друг друга, иначе туннель не поднимется) · `show ip interfaces` — `gre 1` `10.10.10.1/30` `Up` · `ping 10.10.10.2` — `!!!!!`. BR-RTR: `ping 10.10.10.1` — `!!!!!`.
🛠 внешние адреса не пингуются → сделай 8.1 (маршрут по умолчанию и NAT) и вернись · адрес в другой подсети → проверь маску /30.

---

# 7. Динамическая маршрутизация OSPF на HQ-RTR и BR-RTR

Все условия выполняются одним блоком на роутер:
- **7.1** протокол только на интерфейсах туннеля: `ip ospf` включён только в `tunnel gre 1`;
- **7.2** используется OSPF;
- **7.3** маршрутами делятся только друг с другом: единственный интерфейс с OSPF — туннель;
- **7.4** аутентификация, пароль P@ssw0rd.

🖥 HQ-RTR, часть 1 — процесс и сети:
```
configure
router ospf 1
router-id 10.10.10.1
area 0.0.0.0
network 10.10.10.0/30
network 192.100.100.0/25
network 192.100.200.0/26
network 192.100.30.0/29
enable
exit
enable
end
commit
confirm
```
🖥 HQ-RTR, часть 2 — OSPF на туннеле:
```
configure
tunnel gre 1
ip ospf instance 1
ip ospf network point-to-point
ip ospf authentication algorithm cleartext
ip ospf authentication key ascii-text P@ssw0rd
ip ospf
end
commit
confirm
```
🖥 BR-RTR, часть 1:
```
configure
router ospf 1
router-id 10.10.10.2
area 0.0.0.0
network 10.10.10.0/30
network 192.100.20.0/28
enable
exit
enable
end
commit
confirm
```
🖥 BR-RTR, часть 2 — та же, что на HQ-RTR:
```
configure
tunnel gre 1
ip ospf instance 1
ip ospf network point-to-point
ip ospf authentication algorithm cleartext
ip ospf authentication key ascii-text P@ssw0rd
ip ospf
end
commit
confirm
```
🔍 через ~1 минуту: `show ip ospf neighbors` — сосед `Full/PtP` · `show ip ospf interface` — только `gre 1`, `simple password authentication` · HQ-RTR `show ip route ospf` — `192.100.20.0/28` через `gre 1` · BR-RTR `show ip route ospf` — сети 192.100.100.0/25, 192.100.200.0/26, 192.100.30.0/29 · с HQ-SRV `ping -c2 192.100.20.2` — 0% loss · `tracepath -n 192.100.20.2` — второй хоп `10.10.10.2`.
🛠 соседа нет, а туннель пингуется → нет `ttl 64` · `no authentication` → нет строки `algorithm cleartext` · сети LAN не видны → в `router ospf 1` добавь `redistribute connected` · висит в ExStart → подожди 30–60 с, затем в `tunnel gre 1` добавь `ip ospf mtu-ignore` на обоих.

---

# 8. Динамическая трансляция адресов на HQ-RTR и BR-RTR

### 8.1 NAT в сторону ISP для обоих офисов, все устройства выходят в интернет

🖥 HQ-RTR:
```
configure
ip route 0.0.0.0/0 172.16.1.1
nat source
ruleset INET
to interface gigabitethernet 1/0/2
rule 10
action source-nat interface
enable
end
commit
confirm
```
🖥 BR-RTR:
```
configure
ip route 0.0.0.0/0 172.16.2.1
nat source
ruleset INET
to interface gigabitethernet 1/0/1
rule 10
action source-nat interface
enable
end
commit
confirm
```
🔍 роутеры: `ping 8.8.8.8` — `!!!!!` · `show ip route` — `0.0.0.0/0` через 172.16.1.1 / 172.16.2.1 · хосты HQ-SRV, HQ-CLI, BR-SRV: `ping -c2 8.8.8.8` — 0% loss · на ISP `iptables -t nat -L POSTROUTING -n -v` — счётчик MASQUERADE растёт.
🛠 роутер выходит, клиент нет → `show running-config`, блок `nat source`, был ли `confirm`; проверь OSPF-маршруты (7) · роутер не выходит → проверь 2.2 на ISP.

### 8.2 Установка пакетов (служебный подпункт, интернет уже есть)

🖥 HQ-SRV:
```
apt-get update
apt-get install nfs-server bind bind-utils mdadm dosfstools sudo nano
```
🖥 BR-SRV:
```
apt-get update
apt-get install task-samba-dc sudo nano
```
🖥 HQ-CLI:
```
apt-get update
apt-get install task-auth-ad-sssd nfs-clients nano sudo bind-utils
```
🔍 повтор команды — пишет, что всё уже установлено.
🛠 `Couldn't find package` → снова `apt-get update` · очень медленно → терпи, через ESR apt может идти десятки минут; в это время делай 10.

---

# 9. Инфраструктура разрешения имён (DNS на HQ-SRV)

### 9.1 Основной DNS-сервер на HQ-SRV

🖥 HQ-SRV:
```
nano /etc/bind/options.conf
```
Внутри блока `options { … }` сделай так:
- строку `listen-on { 127.0.0.1; };` замени на `listen-on { any; };`
- строку `listen-on-v6 { ::1; };` замени на `listen-on-v6 { none; };`
- добавь (или раскомментируй/поправь, если такие строки уже есть) перед закрывающей `};`:
```
allow-query { any; };
allow-recursion { any; };
dnssec-validation no;
```
Сохрани. Потом:
```
nano /etc/bind/local.conf
```
В конец файла впиши три зоны:
```
zone "au-team.irpo" { type master; file "fwd.db"; };
zone "100.100.192.in-addr.arpa" { type master; file "srv.db"; };
zone "200.100.192.in-addr.arpa" { type master; file "cli.db"; };
```
Сохрани. И обязательно:
```
control bind-caps enabled
```
⚠ Без `control bind-caps enabled` после перезагрузки DNS слушает только 127.0.0.1.
🔍 `cat /etc/bind/local.conf` — три зоны в конце · `named-checkconf` до первого запуска bind может ругнуться на `/etc/bind/rndc.key` (`must have both 'secret' and 'algorithm'`) — это норма, ключ создаётся при первом старте (9.2); после старта `named-checkconf` должен быть пустым.

### 9.2 Разрешение имён в адреса и обратно по таблице 3 (A и PTR)

Записи: hq-rtr A,PTR · br-rtr A · hq-srv A,PTR · hq-cli A,PTR · br-srv A · docker (ISP к HQ-RTR) A · web (ISP к BR-RTR) A. В прямую зону добавлены SRV-записи и `@ A` для домена из п.11.

🖥 HQ-SRV. Файл прямой зоны:
```
nano /etc/bind/zone/fwd.db
```
Впиши:
```
$TTL 1D
@  IN SOA hq-srv.au-team.irpo. root.au-team.irpo. ( 1 12H 1H 1W 1H )
   IN NS hq-srv.au-team.irpo.
@  IN A 192.100.20.2
hq-rtr  IN A 192.100.100.1
br-rtr  IN A 192.100.20.1
hq-srv  IN A 192.100.100.2
hq-cli  IN A 192.100.200.2
br-srv  IN A 192.100.20.2
docker  IN A 172.16.1.1
web     IN A 172.16.2.1
_ldap._tcp               IN SRV 0 0 389  br-srv
_ldap._tcp.dc._msdcs     IN SRV 0 0 389  br-srv
_kerberos._tcp           IN SRV 0 0 88   br-srv
_kerberos._udp           IN SRV 0 0 88   br-srv
_kerberos._tcp.dc._msdcs IN SRV 0 0 88   br-srv
_kpasswd._tcp            IN SRV 0 0 464  br-srv
_kpasswd._udp            IN SRV 0 0 464  br-srv
_gc._tcp                 IN SRV 0 0 3268 br-srv
```
Файл обратной зоны для сети HQ-SRV:
```
nano /etc/bind/zone/srv.db
```
```
$TTL 1D
@  IN SOA hq-srv.au-team.irpo. root.au-team.irpo. ( 1 12H 1H 1W 1H )
   IN NS hq-srv.au-team.irpo.
1  IN PTR hq-rtr.au-team.irpo.
2  IN PTR hq-srv.au-team.irpo.
```
Файл обратной зоны для сети HQ-CLI:
```
nano /etc/bind/zone/cli.db
```
```
$TTL 1D
@  IN SOA hq-srv.au-team.irpo. root.au-team.irpo. ( 1 12H 1H 1W 1H )
   IN NS hq-srv.au-team.irpo.
2  IN PTR hq-cli.au-team.irpo.
```
Права и проверка:
```
cd /etc/bind/zone
chown root:named fwd.db srv.db cli.db
chmod 640 fwd.db srv.db cli.db
named-checkconf
named-checkzone au-team.irpo fwd.db
systemctl enable --now bind
```
🔍 `named-checkzone` — `OK` · `systemctl is-active bind` — `active` · `ss -lnu` — есть `192.100.100.2:53` · `host hq-rtr.au-team.irpo 192.100.100.2` — `192.100.100.1`.
🛠 `named-checkconf` ругается на дубль `dnssec-validation` → удали старую строку в options.conf · ошибка зоны «в строке N» → забыта точка в конце полного имени · rndc.key до первого старта — норма.

### 9.3 Пересылка запросов на внешний DNS 94.232.137.104

🖥 HQ-SRV:
```
nano /etc/bind/options.conf
```
В блок `options { … }` добавь (если закомментированные `//forwarders` есть — используй вместо них):
```
forward only;
forwarders { 94.232.137.104; };
```
Сохрани и примени:
```
named-checkconf
systemctl restart bind
```
🔍 `host ya.ru 192.100.100.2` — адреса.
🛠 SERVFAIL → `dig @94.232.137.104 ya.ru`; сервер не отвечает → временно `forwarders { 8.8.8.8; };`.

### 9.4 Направить хосты на HQ-SRV (служебный подпункт)

🖥 HQ-SRV, HQ-CLI, BR-SRV:
```
nmcli con mod lan ipv4.dns 192.100.100.2
nmcli con mod lan ipv4.dns-search au-team.irpo
nmcli con up lan
```
🖥 HQ-SW: то же, но `mgmt` вместо `lan`.

🔍 HQ-CLI:
- `host hq-rtr.au-team.irpo` — `192.100.100.1`
- `host br-rtr.au-team.irpo` — `192.100.20.1`
- `host hq-srv.au-team.irpo` — `192.100.100.2`
- `host hq-cli.au-team.irpo` — `192.100.200.2`
- `host br-srv.au-team.irpo` — `192.100.20.2`
- `host docker.au-team.irpo` — `172.16.1.1`
- `host web.au-team.irpo` — `172.16.2.1`
- `host 192.100.100.1` — `hq-rtr…` · `host 192.100.100.2` — `hq-srv…` · `host 192.100.200.2` — `hq-cli…`
- `host ya.ru` — адреса

BR-SRV: `host hq-srv.au-team.irpo` — `192.100.100.2`.

---

# 10. Часовой пояс — Москва (все, кроме виртуального коммутатора HQ-SW)

🖥 ISP, HQ-SRV, HQ-CLI, BR-SRV:
```
timedatectl set-timezone Europe/Moscow
```
🖥 HQ-RTR и BR-RTR:
```
configure
clock timezone gmt +3
end
commit
confirm
```
🔍 Linux: `timedatectl` — `Time zone: Europe/Moscow` · `date +%z` — `+0300`. ESR: `show date` — GMT +3. HQ-SW не трогаем.

---

# 11. Контроллер домена Samba DC на BR-SRV

### 11.1 Домен au-team.irpo на BR-SRV

🖥 BR-SRV. Подготовка:
```
rm -f /etc/samba/smb.conf
rm -rf /var/lib/samba
rm -rf /var/cache/samba
mkdir -p /var/lib/samba/sysvol
nano /etc/hosts
```
В конец `/etc/hosts` добавь строку:
```
192.100.20.2 br-srv.au-team.irpo br-srv
```
Создание домена — на вопросы отвечай так:
```
samba-tool domain provision --interactive
```
- Realm: `AU-TEAM.IRPO`
- Domain: `AU-TEAM`
- Server Role: `dc`
- DNS backend: `SAMBA_INTERNAL`
- DNS forwarder IP address: `192.100.100.2`
- Administrator password: `P@ssw0rd` (дважды)

(Если значение в скобках уже верное, достаточно `Enter`.) После этого:
```
systemctl disable --now smb nmb winbind
systemctl enable --now samba
cp -f /var/lib/samba/private/krb5.conf /etc/krb5.conf
```
🔍 `systemctl is-active samba` — `active` · `samba-tool domain info 192.100.20.2` — `Domain: au-team.irpo`.
🛠 provision упал → снова подготовка (rm, rm, rm, mkdir), потом provision.

### 11.2 Ввести в домен HQ-CLI (BR-CLI на схеме нет, пропускаем)

🖥 HQ-CLI:
```
host -t SRV _ldap._tcp.au-team.irpo
system-auth write ad au-team.irpo hq-cli AU-TEAM Administrator 'P@ssw0rd'
```
`Joined …` и следом `DNS update failed!` — норма.
🔍 `host -t SRV …` — `0 0 389 br-srv.au-team.irpo.`.

### 11.3 8 пользователей офиса HQ (hq_cli1 … hq_cli8)

🖥 BR-SRV. Одна команда создаёт одного пользователя:
```
samba-tool user add hq_cli1 'P@ssw0rd'
```
Чтобы не печатать 8 раз, цикл (`$i` не заменяй — это номер):
```
for i in 1 2 3 4 5 6 7 8; do samba-tool user add hq_cli$i 'P@ssw0rd'; done
```
🔍 `samba-tool user list` — в списке `hq_cli1` … `hq_cli8`.

### 11.4 Группа cat_hq, добавить в неё созданных пользователей

🖥 BR-SRV:
```
samba-tool group add cat_hq
for i in 1 2 3 4 5 6 7 8; do samba-tool group addmembers cat_hq hq_cli$i; done
```
🔍 `samba-tool group listmembers cat_hq` — `hq_cli1` … `hq_cli8`.

### 11.5 6 пользователей офиса BR (br_cli1 … br_cli6)

🖥 BR-SRV:
```
for i in 1 2 3 4 5 6; do samba-tool user add br_cli$i 'P@ssw0rd'; done
```
🔍 `samba-tool user list` — `br_cli1` … `br_cli6`.

### 11.6 Группа cat_br, добавить в неё созданных пользователей

🖥 BR-SRV:
```
samba-tool group add cat_br
for i in 1 2 3 4 5 6; do samba-tool group addmembers cat_br br_cli$i; done
```
🔍 `samba-tool group listmembers cat_br` — `br_cli1` … `br_cli6`.

### 11.7 Пользователи cat_hq могут входить на HQ-CLI

🖥 HQ-CLI:
```
nano /etc/sssd/sssd.conf
```
Найди (`Ctrl+W`, слово `access_provider`) строку `access_provider = ad` и замени на две строки:
```
access_provider = simple
simple_allow_groups = cat_hq
```
Сохрани и перезапусти службу:
```
systemctl restart sssd
```
🔍 `cat /etc/sssd/sssd.conf` — видны обе строки · `id hq_cli1` — в группе `cat_hq` · вход под `hq_cli1` / `P@ssw0rd` — успешен.
🛠 `id: no such user` → подожди минуту, `systemctl restart sssd` · `Clock skew` → время на HQ-CLI и BR-SRV должно совпадать (п.10) · нет строки `access_provider = ad` → допиши обе строки руками в секцию `[domain/…]`.

### 11.8 Пользователи cat_br могут входить на BR-CLI

BR-CLI на схеме нет — не выполняем. Проверяем только состав группы (11.6).

### 11.9 cat_hq повышает привилегии только для nano, cat, id

🖥 HQ-CLI. Узнай пути команд:
```
which nano cat id
```
Открой sudoers:
```
control sudo public
nano /etc/sudoers
```
В конец добавь строку (подставь пути из `which`, обычно такие):
```
%cat_hq ALL=(ALL) /usr/bin/nano, /usr/bin/cat, /usr/bin/id
```
Сохрани, проверь, перезагрузи:
```
visudo -c
reboot
```
🔍 после перезагрузки, вход `hq_cli1`: `sudo id` — `uid=0` · `sudo -l` — только nano, cat, id · `sudo ls /root` — отказ · вход `br_cli1` — отказ (не в cat_hq).
🛠 `sudo cat` не пускает → в sudoers укажи оба пути `/bin/cat, /usr/bin/cat` · `which nano` пусто → `apt-get install nano`.

---

# 12. Файловое хранилище на HQ-SRV

### 12.1 Два диска по 1 ГБ → RAID 0 · 12.2 имя md0, конфиг /etc/mdadm.conf

🖥 HQ-SRV:
```
lsblk
```
Выбери два диска по 1 ГБ без разделов (обычно sdb и sdc) и подставь их:
```
mdadm --create /dev/md0 --level=0 --raid-devices=2 /dev/sdb /dev/sdc
mdadm --detail --scan >> /etc/mdadm.conf
```
🔍 `cat /proc/mdstat` — `md0 : active raid0 sdb[…] sdc[…]` · `cat /etc/mdadm.conf` — строка `ARRAY /dev/md0 …`.
🛠 mdadm спросил `Continue?` → `y` · mdadm не найден → `apt-get install mdadm dosfstools`.

### 12.3 Раздел и файловая система FAT32

🖥 HQ-SRV:
```
fdisk /dev/md0
```
В fdisk по порядку: `n`, `p`, `1`, `Enter`, `Enter`, `w`. Затем:
```
mkfs.vfat -F 32 /dev/md0p1
```
🔍 `lsblk` — у `md0` есть `md0p1` около 2 ГБ · `blkid /dev/md0p1` — `TYPE="vfat"`.
🛠 нет `md0p1` → `partprobe /dev/md0`, повтори `mkfs.vfat`.

### 12.4 Автомонтирование в /storage

🖥 HQ-SRV:
```
mkdir /storage
blkid /dev/md0p1
```
Запомни `UUID="…"` (внутри кавычек). Дальше:
```
nano /etc/fstab
```
В конец добавь строку (вместо `ТВОЙ-UUID` впиши UUID):
```
UUID=ТВОЙ-UUID /storage vfat defaults,umask=0000 0 0
```
Сохрани и смонтируй:
```
mount -a
mkdir /storage/share
```
🔍 `df -h` — `/dev/md0p1` смонтирован в `/storage` · `findmnt --verify` — без ошибок.
🛠 `mount -a` ругается → поправь последнюю строку `nano /etc/fstab`; не перезагружайся с битым fstab.

---

# 13. Сетевая файловая система (NFS)

### 13.1 Сервер NFS на HQ-SRV

🖥 HQ-SRV:
```
systemctl enable --now rpcbind
systemctl enable --now nfs-server
```
🔍 `systemctl is-active nfs-server` — `active`.

### 13.2 Общая папка /storage/share — чтение и запись только для сети HQ-CLI

🖥 HQ-SRV:
```
nano /etc/exports
```
В конец добавь строку (старые не трогай):
```
/storage/share 192.100.200.0/26(rw,sync,no_subtree_check,fsid=2)
```
Сохрани и примени:
```
exportfs -ra
exportfs -v
```
(`fsid=2`, потому что в Альте в /etc/exports уже есть `/srv/public` с `fsid=1`.)
🔍 `exportfs -v` — `/storage/share 192.100.200.0/26(rw,…)`.

### 13.3 Автомонтирование на HQ-CLI в /srv/share (BR-CLI нет, пропускаем)

🖥 HQ-CLI:
```
mkdir /srv/share
nano /etc/fstab
```
В конец добавь строку:
```
192.100.100.2:/storage/share /srv/share nfs defaults,_netdev,nofail 0 0
```
Сохрани и проверь:
```
mount -a
touch /srv/share/test
```
🔍 `showmount -e 192.100.100.2` — `/storage/share 192.100.200.0/26` · `df -h` — `192.100.100.2:/storage/share` смонтирован · `touch` без ошибок · на HQ-SRV `ls /storage/share` — файл `test`.
🛠 `access denied by server` → адрес клиента не входит в сеть из exports, на сервере `exportfs -ra` · `Permission denied` → на HQ-SRV в fstab нет `umask=0000`: исправь, `umount /storage`, `mount -a`, `exportfs -ra` · `rpc.statd is not running` → `apt-get install nfs-clients`.

### 13.4 Параметры сервера — в отчёт

Текст для отчёта: «Экспорт `/storage/share` (на RAID0, FAT32); доступ rw только для сети 192.100.200.0/26 (сторона HQ-CLI); параметры sync, no_subtree_check, root_squash (по умолчанию), fsid=2. Клиент монтирует в `/srv/share` через fstab (defaults, _netdev, nofail)».

---

# Итоговая проверка после перезагрузки всех машин

| Пункт | Где | Команда | Должно быть |
|---|---|---|---|
| 1 | все | `hostname` | FQDN с `.au-team.irpo` |
| 5 | HQ-CLI | `ssh sshuser@192.100.100.2` | баннер, 2 попытки |
| 7 | HQ-RTR | `show ip ospf neighbors` | `Full/PtP` |
| 8 | HQ-CLI | `ping -c2 8.8.8.8` | ответы |
| 9 | HQ-CLI | `host hq-rtr.au-team.irpo` | `192.100.100.1` |
| 10 | Linux | `date +%z` | `+0300` |
| 11 | HQ-CLI | `id hq_cli1` | группа `cat_hq` |
| 12 | HQ-SRV | `df -h` | `/storage` смонтирован |
| 13 | HQ-CLI | `df -h` | `/srv/share` смонтирован |
