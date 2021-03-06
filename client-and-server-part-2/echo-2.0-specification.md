# Echo 2.0 Спецификация

Все сообщения представляют собой строки в кодировке UTF-8.

Клиент:

1. Необходимо открыть _публикацию_ `p` и _подписку_ `s` на сервере, используя динамический MDC для _подписки_ , и отправить строку, `HELLO <key>`где `<key>`находится случайное 32-разрядное шестнадцатеричное значение без знака, закодированное в виде строки.



   > Для безопасности шифра Вернама крайне важно, чтобы это значение генерировалось заново при каждой попытке подключения и никогда не использовалось повторно для любой другой связи. Безопасность шифра Вернама может быть мгновенно нарушена путем получения двух разных сообщений, которые были зашифрованы с одним и тем же значением.

2. Необходимо подождать, пока сервер не отправит _строку ответа_ в форме `<session> CONNECT <port> <control-port> <encrypted-session>`или `<session> ERROR <message>`.

* Если ответ имеет форму `<session> ERROR ...`, но `<session>`не соответствует текущему _идентификатору сеанса_ клиента , ответ должен быть проигнорирован, и клиент должен продолжить ожидание.
* Если ответ имеет форму `<session> ERROR <message>`и `<session>`соответствует текущему _идентификатору сеанса_ клиента, клиент должен предположить, что `<message>`это информативное сообщение об ошибке, зарегистрировать сообщение и выйти.
* Если ответ имеет форму `<session> CONNECT ...`, но `<session>`не соответствует текущему _идентификатору сеанса_ клиента , ответ должен быть проигнорирован, и клиент должен продолжить ожидание.
* Если ответ имеет форму `<session> CONNECT <port> <control-port> <encrypted-session>`и `<session>`соответствует текущему _идентификатору сеанса_ клиента :
  * Клиент должен расшифровать `<encrypted-session>`, оценив `k = <encrypted-session> ^ <key>`, где `<key>`это значение из шага 1.
  * Клиент должен создать _подписку_ `t` на порт `<port>`на сервере, используя `<control-port>`для MDC и с явным _идентификатором сеанса_ `k` .
  * Клиент должен создать _публикацию_ `u` для порта `<port>`на сервере с явным _идентификатором сеанса_ `k` .
  * Клиент должен закрыть `s`и `p`, и переходить к шагу 3.
* Если серверу не удается вернуть приемлемый ответ в течение разумного периода времени, клиент должен записать сообщение об ошибке и выйти.

1. Необходимо отправлять сообщения вида `ECHO <message>`к `u`, слушая ответы на `t`.

* Если ответ имеет форму `ERROR <message>`, клиент должен предположить, что `<message>`это информативное сообщение об ошибке, зарегистрировать сообщение и выйти.
* Если ответ имеет форму `ECHO <message>`, клиент должен предположить, что `<message>`это одна из ранее отправленных строк, и вернуться к шагу 3.

Сервер:

1. Необходимо открыть _публикацию,_ `p` которая будет использоваться для отправки начальных сообщений клиентам при их подключении. Он должен предоставлять _контрольный порт,_ чтобы клиенты могли использовать динамический MDC.
2. Необходимо создать _подписку,_ `s` которая будет использоваться для получения первоначальных сообщений от клиентов.

Сервер хранит изначально пустой список _дуологов_ . _Дилогия_ представляет собой структуру , содержащую следующее состояние:

```java
record Duologue(
  InetAddress owner,
  int session_id,
  Publication pub,
  Subscription sub);
```

Когда сообщение получено `s`:

* Если сообщение имеет форму `HELLO <key>`
  * Если размер списка _дуологов_ составляет `n`:
    * Сервер должен написать сообщение вида `<session-id> ERROR server full`к `p`, где `<session-id>`является _идентификатор сеанса_ клиента, отправившего сообщение, и вернуться к ожиданию сообщений.
  * Если хотя бы `m`существуют _дуологи, которым_ принадлежит IP-адрес клиента, отправившего сообщение, то `m`это настраиваемое значение:
    * Сервер должен написать сообщение вида `<session-id> ERROR too many connections for IP address`к `p`, где `<session-id>`является _идентификатор сеанса_ клиента, отправившего сообщение, и вернуться к ожиданию сообщений.
  * Иначе:
    * Сервер должен записать `(a, z, t, u)`в список _дуологов_ , где `t`и `u`только что выделенные _публикации_ и _подписка_ , соответственно, `a`являются IP-адресом клиента и `z`являются недавно выделенным _идентификатором сеанса_ .
    * Сервер должен написать сообщение `<session-id> CONNECT <port> <control-port> <encrypted-session>`в `p`, где `<port>`и `<control-port>`являются номера портов `t`и `u`, `<encrypted-session>`это `z ^ <key>`и `<session-id>`есть _идентификатор сеанса_ клиента, отправившего сообщение.
    * Сервер должен вернуться к ожиданию сообщений.

Когда сообщение получено по подписке `u`о наличии _дилогии_ `i` :

* Если сообщение имеет форму `ECHO <message>`
  * Сервер должен написать сообщение формы `ECHO <message>`к `t`.
* Иначе:
  * Сервер должен написать сообщение формы `ERROR bad message`к `t`.
  * Сервер должен закрыться `t`и `u`.
  * Сервер должен удалить дуолог.

Наконец, сервер должен удалить _дуологи_ , у которых нет подписчиков после истечения произвольно настраиваемого лимита времени.  


