# Справочно-методический материал к лабораторной работе «VLAN, маршрутизация между VLAN»

## 1. Виртуальные локальные сети (VLAN)

### 1.1. Назначение и логика работы
VLAN — это логическая сегментация L2-коммутатора: порты объединяются в изолированные широковещательные домены независимо от физического расположения. Кадры одного VLAN не пересылаются в порты другого VLAN на уровне L2 — для связи между VLAN требуется маршрутизация (L3).

Каждый VLAN идентифицируется номером **1–4094** (1 и 1002–1005 зарезервированы). Имя VLAN задаётся для удобства администрирования.

### 1.2. Создание VLAN и назначение портов
```text
Switch(config)# vlan <НОМЕР>
Switch(config-vlan)# name <ИМЯ>
Switch(config-vlan)# exit

! Назначение порта в VLAN (режим access)
Switch(config)# interface <ИНТЕРФЕЙС>
Switch(config-if)# switchport mode access
Switch(config-if)# switchport access vlan <НОМЕР>
Switch(config-if)# description <ОПИСАНИЕ>
```

* Порт в режиме `access` принадлежит **ровно одному** VLAN и передаёт кадры **без тега**.
* Если порт не назначен явно, он остаётся в VLAN 1 (по умолчанию) — это частая причина «необъяснимой» связности.

### 1.3. Магистральный интерфейс (Trunk) и тегирование 802.1q
Trunk передаёт кадры **нескольких VLAN** по одному физическому линку. Каждый кадр помечается 4-байтовым тегом **802.1q**, который вставляется между полем Source MAC и полем Type/Length:

| Поле тега 802.1q | Размер | Назначение |
|---|---|---|
| **TPID** | 2 байта | Всегда `0x8100` — признак тегированного кадра |
| **PCP** (Priority) | 3 бита | Приоритет QoS (0–7) |
| **DEI** (CFI) | 1 бит | Индикатор допустимости отбрасывания |
| **VID** (VLAN ID) | 12 бит | Номер VLAN (1–4094) |

Поля PCP + DEI + VID вместе образуют **TCI** (Tag Control Information, 16 бит). Для VLAN 10 TCI = `0x000A` = 10 (DEC), для VLAN 20 — `0x0014` = 20 (DEC).

```text
Switch(config)# interface <ИНТЕРФЕЙС>
Switch(config-if)# switchport mode trunk
Switch(config-if)# switchport nonegotiate
Switch(config-if)# switchport trunk allowed vlan <СПИСОК>
Switch(config-if)# switchport trunk native vlan <НОМЕР>
Switch(config-if)# description <ОПИСАНИЕ>
```

* `switchport nonegotiate` — отключает DTP; режим trunk фиксируется статически (рекомендуется в учебных и производственных сетях).
* `switchport trunk allowed vlan 10,20,99` — список разрешённых VLAN; по умолчанию разрешены все.
* **Native VLAN** — VLAN, кадры которого передаются по trunk **без тега**. На обоих концах линка native VLAN должен совпадать, иначе кадры попадут не в тот VLAN (VLAN hopping). По умолчанию native VLAN = 1; в задании используется 99.

> **Важно:** native VLAN должен быть создан на коммутаторе (`vlan 99`), иначе команда `switchport trunk native vlan 99` выдаст ошибку.

### 1.4. Проверка конфигурации VLAN
```text
Switch# show vlan brief
Switch# show vlan id <НОМЕР>
Switch# show interfaces trunk
Switch# show interfaces <ИНТЕРФЕЙС> switchport
```

* `show vlan brief` — список VLAN и портов **access** в каждом (trunk-порты в этой таблице не отображаются как члены VLAN).
* `show interfaces trunk` — режим, native VLAN, список разрешённых и активных VLAN, состояние.
* `show interfaces switchport` — подробный режим порта (administrative/operational), native VLAN, allowed VLAN.

### 1.5. Отключение неиспользуемых портов
```text
Switch(config)# interface range <ДИАПАЗОН>
Switch(config-if-range)# shutdown
```
Используйте `show ip interface brief`, чтобы определить, какие порты не задействованы, и одной командой `interface range` отключите их все.

---

## 2. Таблица MAC-адресов и её связь с VLAN

### 2.1. Как коммутатор изучает MAC
Коммутатор запоминает **MAC-адрес источника** кадра и порт, на который кадр пришёл, в таблице MAC. Запись привязана к паре **(MAC, VLAN)** — в отличие от некоторых старых реализаций, современный Cisco IOS ведёт MAC-таблицу **per-VLAN**.

```text
Switch# show mac address-table
Switch# show mac address-table vlan <НОМЕР>
Switch# show mac address-table interface <ИНТЕРФЕЙС>
Switch# clear mac address-table dynamic
```

### 2.2. Почему один MAC может встречаться несколько раз
Если устройство является шлюзом для **нескольких VLAN** (например, маршрутизатор с сабинтерфейсами 802.1q), его MAC-адрес фигурирует как source в кадрах каждого из этих VLAN. На коммутаторе это даёт **несколько записей с одним MAC, одним интерфейсом, но разными VLAN**:

```
Vlan    Mac Address       Type        Ports
----    -----------       --------    -----
  10    000a.41xx.xxxx    DYNAMIC     Gi0/1   <- MAC R1, VLAN 10
  20    000a.41xx.xxxx    DYNAMIC     Gi0/1   <- MAC R1, VLAN 20 (тот же MAC!)
```

Это не петля и не ошибка: таблица MAC **ведётся отдельно для каждого VLAN**, а R1 — шлюз для обоих VLAN.

> **Чтобы увидеть обе записи**, нужен трафик **в обе стороны** (ping PC0→PC1 **и** PC1→PC0): R1 должен отправить кадр и в VLAN 10, и в VLAN 20.

### 2.3. Широковещательные кадры и границы VLAN
Коммутатор рассылает широковещательный кадр (например, ARP Request) во все порты **того же VLAN**, кроме порта-источника. В порты других VLAN кадр **не уходит** — это и есть изоляция широковещательных доменов. Именно поэтому ARP-запрос из VLAN 10 не достигнет PC1 в VLAN 20.

---

## 3. Маршрутизация между VLAN: Router-on-a-stick

### 3.1. Идея
Один физический интерфейс маршрутизатора делится на **сабинтерфейсы**, каждый из которых обслуживает один VLAN с инкапсуляцией 802.1q. Маршрутизатор принимает тегированные кадры, снимает тег, маршрутизирует пакет и отправляет обратно в нужный сабинтерфейс с новым тегом.

### 3.2. Настройка сабинтерфейсов
```text
Router(config)# interface <ФИЗ_ИНТЕРФЕЙС>.<НОМЕР_VLAN>
Router(config-subif)# encapsulation dot1Q <НОМЕР_VLAN>
Router(config-subif)# ip address <IP> <МАСКА>
Router(config-subif)# description <ОПИСАНИЕ>
```

* Номер сабинтерфейса **не обязан** совпадать с номером VLAN, но совпадение — удобное соглашение.
* `encapsulation dot1Q <VLAN>` обязательна: без неё сабинтерфейс не будет обрабатывать тегированные кадры.
* Физический интерфейс обычно оставляют без IP-адреса (`no ip address`), он работает как «несущая» для сабинтерфейсов.
* Если native VLAN на trunk не 1, на сабинтерфейсе native VLAN указывают `encapsulation dot1Q <VLAN> native`.

### 3.3. Проверка
```text
Router# show ip interface brief
Router# show ip route
Router# show interfaces <ФИЗ_ИНТЕРФЕЙС>.<НОМЕР_VLAN>
```

---

## 4. Маршрутизация между VLAN: Multilayer Switch (SVI)

### 4.1. Идея
Многоуровневый коммутатор (MLS) выполняет маршрутизацию аппаратно. Для каждого VLAN создаётся **SVI** (Switch Virtual Interface) — виртуальный интерфейс, играющий роль шлюза для этого VLAN. Маршрутизация между SVI включается глобально командой `ip routing`.

### 4.2. Настройка SVI
```text
Switch(config)# vlan <НОМЕР>
Switch(config-vlan)# name <ИМЯ>
Switch(config-vlan)# exit

Switch(config)# interface vlan <НОМЕР>
Switch(config-if)# ip address <IP> <МАСКА>
Switch(config-if)# description <ОПИСАНИЕ>
Switch(config-if)# no shutdown

Switch(config)# ip routing
```

* SVI создаётся в момент перехода в `interface vlan <НОМЕР>`; соответствующий VLAN должен существовать.
* `ip routing` обязательна — без неё MLS не маршрутизирует между SVI (остаётся L2-коммутатором).
* SVI активен (up/up), только если в VLAN есть хотя бы один порт в состоянии up (или есть другой активный SVI в этом VLAN).

### 4.3. L3-интерфейс (`no switchport`)
Порт MLS можно перевести в режим L3 — он перестаёт быть коммутируемым и получает IP-адрес:
```text
Switch(config)# interface <ИНТЕРФЕЙС>
Switch(config-if)# no switchport
Switch(config-if)# ip address <IP> <МАСКА>
Switch(config-if)# description <ОПИСАНИЕ>
Switch(config-if)# no shutdown
```

### 4.4. Статическая маршрутизация между MLS и маршрутизатором
```text
Router(config)# ip route <СЕТЬ> <МАСКА> <NEXT_HOP>
Switch(config)# ip route <СЕТЬ> <МАСКА> <NEXT_HOP>
```
Для связности «туда и обратно» маршруты нужны **на обоих устройствах**: R1 должен знать сети за MLS, MLS — сети за R1.

### 4.5. Проверка
```text
Switch# show ip interface brief
Switch# show ip route
Switch# show interfaces <ИНТЕРФЕЙС> switchport
```
`show ip route` должен содержать connected-сети SVI и L3-интерфейса, а также статические маршруты (`S`).

---

## 5. Сводная таблица команд верификации

| Задача | Команда Cisco IOS |
|---|---|
| Список VLAN и портов access | `show vlan brief` |
| Информация о конкретном VLAN | `show vlan id <НОМЕР>` |
| Состояние trunk-интерфейсов, native и allowed VLAN | `show interfaces trunk` |
| Подробный режим порта (access/trunk, native, allowed) | `show interfaces <ИНТЕРФЕЙС> switchport` |
| Таблица MAC-адресов | `show mac address-table` |
| Таблица MAC по VLAN / интерфейсу | `show mac address-table vlan <НОМЕР>` / `interface <ИНТЕРФЕЙС>` |
| Очистка динамических записей MAC | `clear mac address-table dynamic` |
| Краткий статус интерфейсов и IP | `show ip interface brief` |
| Таблица маршрутизации | `show ip route` |
| Статус сабинтерфейса / SVI | `show interfaces <ИНТЕРФЕЙС>` |

---

## 6. Типичные ошибки и «грабли»

1. **Забыли создать VLAN** (особенно native 99) — `switchport access vlan` / `switchport trunk native vlan` выдаёт ошибку или порт остаётся в VLAN 1.
2. **Несовпадение native VLAN** на двух концах trunk — трафик native VLAN уходит в чужой VLAN (VLAN hopping), часть трафика «теряется».
3. **DTP не отключён** — trunk может не подняться или подняться в неожиданном режиме; используйте `switchport nonegotiate` и статический `switchport mode trunk`.
4. **Забыли `ip routing`** на MLS — SVI есть, IP настроены, но ping между VLAN не идёт.
5. **SVI down/down** — в VLAN нет активных портов; проверьте `show vlan` и состояние портов.
6. **Забыли `encapsulation dot1Q`** на сабинтерфейсе — сабинтерфейс не обрабатывает тегированные кадры.
7. **Односторонняя статика** — маршрут есть только на одном устройстве; ping в одну сторону работает, в обратную — нет.
8. **MAC R1 «дважды»** — это не петля: таблица MAC ведётся per-VLAN, R1 — шлюз для двух VLAN. Чтобы увидеть обе записи, нужен трафик в обе стороны.
9. **`show vlan brief` не показывает trunk-порты** как членов VLAN — это нормально; для trunk используйте `show interfaces trunk`.
10. **Забыли `write memory`** — конфигурация потеряется после перезагрузки.
