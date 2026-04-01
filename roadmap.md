# Roadmap по подготовке
## Сети
### Cisco CNA

https://youtube.com/playlist?list=PLHpWwR6fSBN5gTzVFoNbKfYi7ziZfxjwf&si=hKVaLtH0a9X64dqm

|1|2 |2  |3  |4  |5  |
| --- | --- | --- | --- | --- | --- |
| 01.04 | 01.04 | 01.04 | 01.04 |  |  |
|6  |7  |8  |9  |10  |11  |
|  |  |  |  |  |  |
|12  |13  |14  |15  |16  |17  |
|  |  |  |  |  |  |
|18  |19  |20  |21  |22  |23  |
|  |  |  |  |  |  |
|24  |25  |26  |27 |28  |29  |
|  |  |  |  |  |  |
|30  |31  |32  |33  |34  |35  |
|  |  |  |  |  |  |
|36  |37  |38  |39  |  |  |
|  |  |  |  |  |  |
### СДСМ
https://linkmeup.ru/blog/1190/
|1|2 |2  |3  |4  |5  |
| --- | --- | --- | --- | --- | --- |
| 01.04 | 01.04 |  |  |  |  |
|6  |7  |8  |9  |10  |11  |
|  |  |  |  |  |  |
|12  |13  |14  |15  |  |  |
|  |  |  |  |  |  |
## Linux

### Iptables
| # | Тема | Что нужно понять | Практика |
|---|------|------------------|----------|
| 1.1 | **Таблицы и цепи** | `filter` (INPUT/OUTPUT/FORWARD) — для фильтрации; `nat` (PREROUTING/POSTROUTING) — для NAT; `mangle` — для изменения пакетов; `raw` — для bypass connection tracking | Выполнить `iptables -L -t filter` и `iptables -L -t nat`. Запомнить, какая таблица за что отвечает |
| 1.2 | **Порядок обработки** | Правила читаются сверху вниз, применяется первое совпадение. Если ни одно не подошло — срабатывает политика по умолчанию (ACCEPT/DROP) | Создать 3 правила с разными портами, выполнить `iptables -L -nv --line-numbers`, удалить правило по номеру |
| 1.3 | **Подготовка среды** | UFW — надстройка над iptables. Для чистого понимания нужно отключить | `sudo ufw disable`; `sudo apt install iptables iptables-persistent`; `sudo systemctl stop ufw` |
| 1.4 | **Базовые команды** | `-L` (list), `-F` (flush), `-P` (policy), `-A` (append), `-D` (delete), `-I` (insert), `-R` (replace), `-n` (no DNS) | Очистить все правила: `iptables -F`; установить политики; добавить и удалить правило |

| # | Тема | Что нужно понять | Практика | 
|---|------|------------------|----------|
| 2.1 | **Политики по умолчанию** | DROP — безопасно, но требует явного разрешения; ACCEPT — открыто, но требует явного запрета. Для сервера: INPUT DROP, OUTPUT ACCEPT | `iptables -P INPUT DROP`; `iptables -P FORWARD DROP`; `iptables -P OUTPUT ACCEPT`. Проверить, что SSH не отвалился | 
| 2.2 | **Loopback и состояния** | lo (127.0.0.1) всегда разрешать. conntrack отслеживает состояния: ESTABLISHED, RELATED, NEW, INVALID | `iptables -A INPUT -i lo -j ACCEPT`; `iptables -A INPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT` |
| 2.3 | **Открытие портов** | Указывать протокол (tcp/udp), порт и состояние. Для сервисов — разрешать только NEW | SSH: `iptables -A INPUT -p tcp --dport 22 -m conntrack --ctstate NEW,ESTABLISHED -j ACCEPT`. HTTP/HTTPS аналогично |
| 2.4 | **Фильтрация по IP** | `-s` (source), `-d` (destination). Комбинация с портами | Разрешить конкретный IP: `iptables -A INPUT -s 192.168.1.10 -j ACCEPT`; заблокировать: `iptables -A INPUT -s 10.0.0.5 -j DROP` | | 
| 2.5 | **ICMP (ping)** | Типы ICMP: echo-request (8), echo-reply (0). Для безопасности разрешать только нужные | `iptables -A INPUT -p icmp --icmp-type echo-request -j ACCEPT`; `iptables -A INPUT -p icmp --icmp-type echo-reply -j ACCEPT` |

| # | Тема | Что нужно понять | Практика |
|---|------|------------------|----------|
| 3.1 | **conntrack детально** | NEW — новое соединение; ESTABLISHED — активное; RELATED — связанное (FTP data, ICMP error); INVALID — битые пакеты (DROP) | Написать правила для FTP с RELATED: `-p tcp --dport 21 -m conntrack --ctstate NEW,ESTABLISHED -j ACCEPT` |
| 3.2 | **multiport** | Экономия правил — объединение портов | Открыть 22,80,443 одной строкой: `-m multiport --dports 22,80,443 -m conntrack --ctstate NEW -j ACCEPT` |
| 3.3 | **iprange** | Работа с диапазонами, а не подсетями | `-m iprange --src-range 192.168.1.100-192.168.1.200 -j DROP` |
| 3.4 | **limit (анти-DoS)** | Ограничение частоты пакетов. `--limit` — скорость, `--limit-burst` — начальный лимит | ICMP не более 6 в минуту: `-m limit --limit 6/minute --limit-burst 3 -j ACCEPT`; остальные DROP |
| 3.5 | **string и time** | Фильтрация по содержимому пакета; по времени суток | Блокировать DNS запросы к вредоносным доменам: `-m string --string "bad.com" --algo bm -j DROP`; разрешить доступ только в рабочие часы |

| # | Тема | Что нужно понять | Практика |
|---|------|------------------|----------|
| 4.1 | **IP Forwarding** | Ядро Linux может работать как роутер. Нужно включить форвардинг | `sudo sysctl -w net.ipv4.ip_forward=1`; добавить в `/etc/sysctl.conf`: `net.ipv4.ip_forward=1` |
| 4.2 | **SNAT / MASQUERADE** | Source NAT — меняет IP источника. MASQUERADE автоматически подставляет IP интерфейса | Для локальной сети: `iptables -t nat -A POSTROUTING -s 192.168.1.0/24 -o eth0 -j MASQUERADE`. Для статического IP: `-j SNAT --to-source 1.2.3.4` |
| 4.3 | **DNAT (порт-форвардинг)** | Destination NAT — перенаправление портов на внутренние сервера | Проброс 8080 на внутренний 80: `iptables -t nat -A PREROUTING -p tcp --dport 8080 -j DNAT --to-destination 192.168.1.100:80` |
| 4.4 | **REDIRECT** | Локальное перенаправление (для прокси) | Весь HTTP трафик на Squid (3128): `iptables -t nat -A PREROUTING -p tcp --dport 80 -j REDIRECT --to-port 3128` |

| # | Тема | Что нужно понять | Практика |
|---|------|------------------|----------|
| 5.1 | **Сохранение правил** | iptables правила не сохраняются после перезагрузки. Нужны инструменты persistence | `sudo netfilter-persistent save`; `sudo iptables-save > /etc/iptables/rules.v4`. Проверить загрузку при старте |
| 5.2 | **Логирование** | LOG перед DROP для анализа. Важно не заспамить логи | `iptables -A INPUT -j LOG --log-prefix "DROP: " --log-level 4`. Проверить: `tail -f /var/log/syslog \| grep "DROP:"` |
| 5.3 | **Отладка** | Просмотр счетчиков пакетов, трассировка | `iptables -L -nv` (счетчики); `iptables -t nat -L -nv`; `conntrack -L` (активные соединения) |
| 5.4 | **IPv6 (ip6tables)** | Если не используете IPv6 — закрыть | `ip6tables -P INPUT DROP`; `ip6tables -P FORWARD DROP`; сохранить аналогично |

| # | Сценарий | Что нужно реализовать |
|---|----------|----------------------|
| 6.1 | **Безопасный веб-сервер** | Разрешить: SSH (22), HTTP (80), HTTPS (443). Запретить всё остальное. Включить логирование подозрительных попыток. Ограничить SSH по IP (если есть статический) |
| 6.2 | **NAT + Firewall для локальной сети** | Сервер как роутер: LAN (192.168.1.0/24) выходит в интернет через MASQUERADE. Разрешить LAN доступ только к определенным портам (80,443) |
| 6.3 | **Проброс портов** | Перенаправить внешний порт 2222 на внутренний 22 на другой сервер. Добавить ограничение по IP источника для этого проброса |
| 6.4 | **Защита от DoS** | Ограничить SYN пакеты (10/сек) на порт 80: `-m limit --limit 10/second -j ACCEPT`. Использовать `-m conntrack --ctstate INVALID -j DROP` |
### Смена порта ssh
|1|2 |2  |3  |4  |5  |
| --- | --- | --- | --- | --- | --- |
|  |  |  |  |  |  |
|6  |7  |8  |9  |10  |11  |
|  |  |  |  |  |  |
|12  |13  |14  |15  |16  |17  |
|  |  |  |  |  |  |
### NAT
|1|2 |2  |3  |4  |5  |
| --- | --- | --- | --- | --- | --- |
|  |  |  |  |  |  |~~~~
## IaC
### Модель зрелости

### Ansible
### Docker


## Прочитанная литература

![Основы компьютерных сетей](.\images\book1.png)
Олифер - Основы компьютерных сетей (в процессе прочтения)
![Основы компьютерных сетей](.\images\book2.png)
Кочер - Микросервисы и контейнеры Docker
![Основы компьютерных сетей](.\images\book3.png)
Титмус - Облачный Go
