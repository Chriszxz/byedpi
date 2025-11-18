Implementation of some DPI bypass methods.
The program is a local SOCKS proxy server.

Usage example:
```
ciadpi --disorder 1 --auto=torst --tlsrec 1+s
ciadpi --fake -1 --ttl 8
```

------
### Описание аргументов
```
-i, --ip <ip>
Listening IP, default 0.0.0.0

-p, --port <num>
Listening port, default 1080

-D, --daemon
Run in daemon mode
Only supported on Linux and BSD systems

-w, --pidfile <filename>
Location of the PID file

-E, --transparent
Run in transparent proxy mode; SOCKS will not work

-c, --max-conn <count>
Maximum number of client connections, default 512

-I, --conn-ip <ip>
Address to which outgoing connections will be bound, default ::
If an IPv4 address is specified, IPv6 requests will be rejected

-b, --buf-size <size>
Maximum size of data received and sent in a single recv/send call
The size is specified in bytes; default is 16384

-g, --def-ttl <num>
TTL value for all outgoing connections
Can be useful for bypassing non-standard/reduced TTL detection.

-N, --no-domain
Discard requests if a domain is specified as the address.
Because Since resolving is performed synchronously, it can slow down or even freeze the operation.

-U, --no-udp
Do not proxy UDP

-F, --tfo
Enables TCP Fast Open
If the server supports it, the first packet will be sent immediately along with the SYN
Supported only on Linux (4.11+)

-A, --auto <t,r,s,n>
Automatic mode
If an event similar to a blocking or crash occurs,
the bypass parameters following this option will be applied.
Possible events:
torst: Timeout expired or the server dropped the connection after the first request
redirect: HTTP Redirect with a Location whose domain does not match the outgoing one
ssl_err: No ServerHello was received in response to ClientHello, or the SH contains an invalid session_id
none: The previous group was skipped, for example due to domain or protocol restrictions

-L, --auto-mode <0-3>
0: Cache IP only if a reconnection is possible
1: Cache IP also if:
torst - timeout/connection dropped during packet exchange (i.e., after the first data from the server)
ssl_err - only one round trip (request-response/request-response-request) has occurred
2: Sort groups by the number of trigger occurrences, from lowest to highest
3: 1 and 2 simultaneously

-u, --cache-ttl <sec>
Cache lifetime, default 100800 (28 hours)

-y, --cache-dump <file|->
Dump cache to file or stdout. Format: <ip> <port> <group index> <time> <host>

-T, --timeout <sec>
Timeout for the first response from the server in seconds
In Linux, this is converted to milliseconds, so you can specify a fractional number.

-K, --proto <t,h,u,i>
Protocol whitelist: tls,http,udp,ipv4

-H, --hosts <file|:string>
Limit the scope of parameters to a list of domains
Domains must be separated by a newline or a space.

-j, --ipset <file|:str>
Limit by specific IPs/subnets

-V, --pf <port[-portr]>
Limit by ports

-R, --round <num[-numr]>
Which requests to apply obfuscation to
Defaults to 1, i.e. to the first request

-s, --split <pos_t>
Split the request by the specified position
The position is of the form offset[:repeats:skip][+flag1[flag2]]
Flags:
+s: add SNI offset
+h: add Host offset
+n: zero offset
Additional flags:
+e: end; +m: middle
Examples:
0+sm - split the request in the middle of the SNI
1:3:5 - split by positions 1, 6, and 11
The key can be specified multiple times to split the request into multiple positions
If offset is negative and has no flags, the packet size is added to it.

-d, --disorder <pos_t>
Similar to --split, but the parts are sent in reverse order.

-o, --oob <pos_t>
Similar to --split, but the part is sent as out-of-bounds data.

-q, --disoob <pos_t>
Similar to --disorder, but the part is sent as out-of-bounds data.

-f, --fake <pos_t>
Similar to --disorder, except that a part of the fake part is sent before the first part.
The number of bytes sent from the fake part is equal to the size of the part being split.
! May be unstable on Windows.

-t, --ttl <num>
TTL for the fake packet, defaults to 8
You need to choose a value so that the packet doesn't reach the server, but is processed by DPI.

-S, --md5sig
Set the TCP MD5 Signature option for the fake packet.
Most servers (mainly Linux-based) discard packets with this option.
Only supported on Linux; may be disabled in some kernel builds (< 3.9, Android).

-O, --fake-offset <pos_t>
Offset the start of the fake data.
Offsets with flags are calculated relative to the original request.

-l, --fake-data <file|:str>
Specify your own fake packets.
The string may contain escape characters (\n,\0,\0x10).

-e, --oob-data <char>
Byte sent outside the main stream, defaults to 'a'.
Can be specified.
```

------
### Подробнее
`--split`

Разбивает запрос на части. Пример на запросе в 30 байт:
- Параметры: `--split 3 --split 7`
- Порядок отправки: 1-3, 3-7, 7-30  

Позиции следует указывать в порядке возрастания.  

------
`--disorder`

Часть, попадающая под disorder, будет отправлена с TTL=1, т.е. фактически не будет никуда доставлена.
ОС узнает об этом лишь после отсылки последующей части, когда сервер сообщит о потере с помощью SACK.
Системе придется отослать предыдущий пакет заново, тем самым нарушив обычный порядок.
- Параметры: `--disorder 7`
- Порядок отправки: 7-30, 1-7  

Вышесказанное распространяется только на Linux.
В Windows ретрансмиссия начинается с позиции, с которой начались потери (максимальный ACK, полученный от сервера):
- Параметры: `--disorder 7`
- Порядок отправки: 7-30, 1-30

Поэтому желательно использовать ещё и `split`:  
- Параметры: `--split 7 --disorder 23`
- Порядок отправки: 1-7, 23-30, 7-30

На практике оптимально использовать:  
* Linux: `--disorder 1`
* Windows: `--split 1+s --disorder 3+s`

------
`--fake`

- Параметры: `--fake 7`
- Порядок отправки: 1-7 фейк, 7-30 оригинал, 1-7 оригинал

Данные в первой части запроса заменяются на поддельные.  
Эта часть должна пройти через DPI, но не дойти до сервера.
А раз часть не дойдет, то ОС отправит ее снова, тем самым изменив порядок подобно `disorder`.
Для того, чтобы фейк не дошел до сервера, есть опции `ttl` и `md5sig`.  

TTL необходимо подбирать такой, чтобы пакет прошел через все DPI, но не дошел до сервера.  
Для Linux есть md5sig. Он устанавливает опцию TCP MD5 Signature, что не дает пакету быть принятым многими серверами.
К сожалению, md5sig работает не во всех сборках.  

Для Windows есть еще один способ избежать обработки фейка сервером.
Это комбинирование `fake` с `disorder`:
- Параметры: `--disorder 1 --fake 7`
- Порядок отправки: 2-7 фейк, 7-30 оригинал, 1-30 оригинал  

Если поддельный пакет и дойдет до сервера, то он будет перезаписан из-за полной ретрансмисси.  

На практике оптимально использовать:  
* Linux: `--fake -1 --md5sig`
* Windows: `--disorder 1 --fake -1`

------
`--oob`

TCP может отсылать данные вне основного потока, используя флаг URG, однако лишь 1 байт в пакете.  
Все данные в таком пакете будут доставлены приложению, кроме последнего байта, который и является внеканальным:
- Параметры: `--oob 3`
- Отправка: 1-4 с флагом URG (1-3 данные запроса + 4-й байт, который будет усечен), 3-30

Этот байт желательно помещать в SNI: `--oob 3+s` 

------
`--disoob`

Схож с `--disorder`, но часть отправляется с OOB байтом:
- Параметры: `--disoob 3`
- Отправка: 3-30, 1-4 с флагом URG (1-3 данные запроса + 4-й байт, который будет усечен)

При использовании с `--fake` или `--disorder` можно получить пакет, где OOB байт будет находиться на месте разбиения:
- Параметры: `--disoob 3 --disorder 7`
- Отправка: 3-30, 1-8 с флагом URG (1-3 + байт который будет усечен + 4-8)

------
`--tlsrec`

Одну TLS запись можно разбить на несколько, немного переделав заголовок.  
На месте разбиения вставляется новый заголовок, увеличивая размер запроса на 5 байт.  

Этот заголовок можно поместить в середину SNI, не давая возможность DPI правильно его прочитать: 
`--tlsrec 3+s`

Хоть `tlsrec` и `oob` запутывают DPI, они также могут запутать всякие мидлбоксы, которые не поддерживают полноценный стек TCP/TLS.  
Из-за этого их следует использовать вместе с `--auto`:  
`--auto=torst --timeout 3 --tlsrec 3+s`  
В примере `tlsrec` будет применяться лишь в случаях, когда сброшено подключение или вышел таймаут, т.е. когда, скорее всего, произошла блокировка.  
Можно наоборот - отменять tlsrec, если сервер сбрасывает подключение или откидывает пакет:  
`--tlsrec 3+s --auto=torst --timeout 3`  

------
`-Y, --drop-sack`

Заставляет ядро игнорировать пакеты с расширением TCP SACK.
Это расширение позволяет подтверждать получение отдельных сегментов данных.
Если первая часть запроса будет потеряна, а до сервера дойдет лишь вторая, то сервер с помощью этого расширения может уведомить клиента об этом. Тогда клиент, зная, что вторая часть дошла, отправит лишь первую.  
Зачем игнорировать это расширение? Второй сегмент может быть фейковым. Если он дойдет до сервера, но клиент об этом не узнает, то он попытается переотправить его. Однако этот сегмент будет содержать уже оригинальные данные, которые перезапишут фейковые, тем самым предотвратив поломку протокола.  
Так как быстрое подтверждение работать не будет, то это сломает `disorder`, а также добавит задержку перед ретрансмиссией (около 200ms).

------
`--auto`, `--hosts`

Параметр `auto` делит опции на группы.
Для каждого запроса они обходятся слева на право.
Сначала проверяется триггер, указанный в `auto`, затем `pf`, `ipset`, `proto` и `hosts`.

Можно указывать несколько групп опций, раделяя их данным параметром.  
Параметры, которые идут ниже `--timeout` в help-тексте, можно вынести в отдельную группу.  

#### Примеры:
```
--fake -1 --ttl 10 --auto=ssl_err --fake -1 --ttl 5
```
По умолчанию использовать `fake` с ttl=10, в случае ошибки использовать `fake` с ttl=5

```
--hosts list.txt --disorder 3 --auto=none
```
Применять запутывание только для доменов из list.txt

```
--hosts list.txt --auto=none --disorder 3
```
Не применять запутывание для доменов из list.txt

```
--auto=torst --hosts list.txt --disorder 3
```
По умолчанию ничего не делать, использовать disorder при условии, что произошла блокировка и домен входит в list.txt.

```
--proto=http,tls --disorder 3 --auto=none
```
Запутывать только HTTP и TLS

```
--proto=http --fake -1 --fake-data=':GET /...' --auto=none --fake -1
```
Переопределить фейковый пакет для HTTP

------
### Сборка
Для сборки понадобится: 
`make`, `gcc/clang` для Linux, `mingw` для Windows  

* Linux: `make`
* Windows: `make windows CC=x86_64-w64-mingw32-gcc`

------
### Docker образ

Docker образ выкладывается на [DockerHub](https://hub.docker.com/r/hufrea/byedpi).
Пример конфигурации контейнера можно найти в [dist/docker](dist/docker).

------
### Дополнительная информация о DPI, источники идей  
* https://github.com/bol-van/zapret/blob/master/docs/readme.md  
* https://geneva.cs.umd.edu/papers/geneva_ccs19.pdf  
* https://habr.com/ru/post/335436  
