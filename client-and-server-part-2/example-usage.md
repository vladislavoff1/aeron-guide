# Пример использования

Мы запускаем экземпляр сервера `apricot`, привязывая его к портам `9000`и `9001`по адресу `5.6.7.8`. Сервер начнет присваивать клиентские _дуологии_ через порт UDP `9010`и ограничен максимумом `4`клиентов:

```text
apricot$ java -cp aeron-guide.jar com.io7m.aeron_guide.take2.EchoServer /tmp/aeron-server 5.6.7.8 9000 9001 9010 4
```

Мы запускаем экземпляр клиента `maize`, приказывая ему подключиться `apricot`и немедленно отключить его \(для краткости\) после того, как он отправил и получил одно `ECHO`сообщение:

```text
maize$ java -cp aeron-guide.jar com.io7m.aeron_guide.take2.EchoClient /tmp/aeron-client 5.6.7.8 9000 9001
DEBUG [main] com.io7m.aeron_guide.take2.EchoClient: initial subscription connected
DEBUG [main] com.io7m.aeron_guide.take2.EchoClient: initial publication connected
TRACE [main] com.io7m.aeron_guide.take2.EchoMessages: [-555798973] send: HELLO 611786D9
DEBUG [main] com.io7m.aeron_guide.take2.EchoClient: waiting for response
TRACE [main] com.io7m.aeron_guide.take2.EchoClient: [-555798973] response: -555798973 CONNECT 9014 9011 129F3C18
DEBUG [main] com.io7m.aeron_guide.take2.EchoClient: [-555798973] connect 9014 9011 (encrypted 312425496)
DEBUG [main] com.io7m.aeron_guide.take2.EchoClient: CONNECT subscription connected
DEBUG [main] com.io7m.aeron_guide.take2.EchoClient: CONNECT publication connected
TRACE [main] com.io7m.aeron_guide.take2.EchoMessages: [1938340545] send: ECHO 12cbcf2e9c84b2f0
DEBUG [main] com.io7m.aeron_guide.take2.EchoClient: [-555798973] response: ECHO 12cbcf2e9c84b2f0
DEBUG [main] com.io7m.aeron_guide.take2.EchoClient: [-555798973] ECHO 12cbcf2e9c84b2f0
```

Мы видим, что, несмотря на тот факт, что `maize`за маршрутизатором NAT `tomato`он все еще может подключаться и получать ответы от `apricot`. Адрес `maize`\( `10.10.0.5`\) с точки зрения, `apricot`кажется, адрес `tomato`\( `10.10.0.1`\):

```text
(apricot)

DEBUG [com.io7m.aeron_guide.take2.server[16]] com.io7m.aeron_guide.take2.EchoServer: [-555798973] initial client connected (10.10.0.1:55297)
DEBUG [com.io7m.aeron_guide.take2.server[16]] com.io7m.aeron_guide.take2.EchoServer: [-555798973] received: HELLO 611786D9
DEBUG [com.io7m.aeron_guide.take2.server[16]] com.io7m.aeron_guide.take2.EchoServerDuologue: creating new duologue at /5.6.7.8 (9014,9011) session 1938340545 for /10.10.0.1
DEBUG [com.io7m.aeron_guide.take2.server[16]] com.io7m.aeron_guide.take2.EchoServer: [-555798973] created new duologue
TRACE [com.io7m.aeron_guide.take2.server[16]] com.io7m.aeron_guide.take2.EchoMessages: [-2147483648] send: -555798973 CONNECT 9014 9011 129F3C18
DEBUG [com.io7m.aeron_guide.take2.server[16]] com.io7m.aeron_guide.take2.EchoServer: [-555798973] initial client disconnected (10.10.0.1:55297)
DEBUG [com.io7m.aeron_guide.take2.server[16]] com.io7m.aeron_guide.take2.EchoServerDuologue: [1938340545] client with correct IP connected
DEBUG [com.io7m.aeron_guide.take2.server[16]] com.io7m.aeron_guide.take2.EchoServerDuologue: [1938340545] received: ECHO 12cbcf2e9c84b2f0
TRACE [com.io7m.aeron_guide.take2.server[16]] com.io7m.aeron_guide.take2.EchoMessages: [1938340545] send: ECHO 12cbcf2e9c84b2f0
DEBUG [com.io7m.aeron_guide.take2.server[16]] com.io7m.aeron_guide.take2.EchoServerDuologue: [1938340545] last client (/10.10.0.1) disconnected
DEBUG [com.io7m.aeron_guide.take2.server[16]] com.io7m.aeron_guide.take2.EchoServer: [-555798973] duologue expired
DEBUG [com.io7m.aeron_guide.take2.server[16]] com.io7m.aeron_guide.take2.EchoServer: [-555798973] duologue closed
DEBUG [com.io7m.aeron_guide.take2.server[16]] com.io7m.aeron_guide.take2.EchoServer: [-555798973] deleted duologue
```

Журналы фильтра пакетов `tomato`четко показывают дейтаграммы, проходящие через:

```text
rule 1/(match) pass in on em1: 10.10.0.5.43645 > 5.6.7.8.9001: udp 36 (DF)
rule 1/(match) pass out on em0: 10.10.0.1.50088 > 5.6.7.8.9001: udp 36 (DF)
rule 1/(match) pass in on em1: 10.10.0.5.54239 > 5.6.7.8.9000: udp 40 (DF)
rule 1/(match) pass out on em0: 10.10.0.1.55297 > 5.6.7.8.9000: udp 40 (DF)
rule 1/(match) pass in on em1: 10.10.0.5.49564 > 5.6.7.8.9011: udp 36 (DF)
rule 1/(match) pass out on em0: 10.10.0.1.60760 > 5.6.7.8.9011: udp 36 (DF)
rule 1/(match) pass in on em1: 10.10.0.5.36996 > 5.6.7.8.9014: udp 40 (DF)
rule 1/(match) pass out on em0: 10.10.0.1.60643 > 5.6.7.8.9014: udp 40 (DF)
```

Запуск трех клиентских экземпляров, `maize`а затем попытка запустить четвертый показывает, что сервер больше не будет разрешать подключения `maize`с IP-адреса:

```text
maize$ java -cp aeron-guide.jar com.io7m.aeron_guide.take2.EchoClient /tmp/aeron-client4 5.6.7.8 9000 9001
DEBUG [main] com.io7m.aeron_guide.take2.EchoClient: initial subscription connected
DEBUG [main] com.io7m.aeron_guide.take2.EchoClient: initial publication connected
TRACE [main] com.io7m.aeron_guide.take2.EchoMessages: [-2147483648] send: HELLO 16418C02
DEBUG [main] com.io7m.aeron_guide.take2.EchoClient: waiting for response
TRACE [main] com.io7m.aeron_guide.take2.EchoClient: [-2147483648] response: -2147483648 ERROR too many connections for IP address
ERROR [main] com.io7m.aeron_guide.take2.EchoClient: [-2147483648] server returned an error: too many connections for IP address
Exception in thread "main" com.io7m.aeron_guide.take2.EchoClientRejectedException: Server rejected this client
    at com.io7m.aeron_guide.take2.EchoClient.waitForConnectResponse(EchoClient.java:353)
    at com.io7m.aeron_guide.take2.EchoClient.run(EchoClient.java:201)
    at com.io7m.aeron_guide.take2.EchoClient.main(EchoClient.java:165)
```

