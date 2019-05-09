# Echo Сервер

Дизайн сервера не сильно отличается от дизайна клиента. Основные различия заключаются в том, когда и где создаются публикации и подписки. Клиент просто открывает пару публикация/подписка для единственного сервера, с которым он будет взаимодействовать, и поддерживает их до тех пор, пока пользователь его не завершит. Сервер, однако, должен создавать публикации и / или подписки в ответ на подключение клиентов и должен иметь возможность обращаться к клиентам индивидуально, чтобы отправлять ответы.

Мы структурируем сервер `EchoServer` аналогично `EchoClient` включая статический метод `create`который настраивает Aeron и медиа-драйвер. Единственное отличие состоит в том, что у сервера нет _удаленного адреса_ как части его информации о конфигурации; он указывает только локальный адрес, к которому будут подключаться клиенты.

Метод `run` , однако, отличается в нескольких аспектах.

[EchoServer.setupSubscription \(\)](https://github.com/io7m/aeron-guide/blob/f967e09d36ec8c7f525a66ade2f6f00ab43dcd51/src/main/java/com/io7m/aeron_guide/take1/EchoServer.java#L301)

```java
  private Subscription setupSubscription() {
    final String sub_uri =
      new ChannelUriStringBuilder()
        .reliable(TRUE)
        .media("udp")
        .endpoint(this.local_address.toString().replaceFirst("^/", ""))
        .build();

    LOG.debug("subscription URI: {}", sub_uri);
    return this.aeron.addSubscription(
      sub_uri,
      ECHO_STREAM_ID,
      this::onClientConnected,
      this::onClientDisconnected);
  }
```

_Настроенная_ сервером _подписка_ дополняется парой обработчиков _Image_ . _Image_ в терминологии Aeron - это репликация потока _публикации_ на стороне _подписки_ . Другими словами, когда клиент создает _публикацию_ для общения с сервером, сервер получает _image_ _публикации_ клиента, которое содержит _подписку,_ из которой сервер может читать. Когда клиент пишет сообщение в свою _публикацию_ , сервер может прочитать сообщение из _подписки_ в _образе_.

Когда _image_ становится доступным, это означает, что клиент подключился. Когда _image_ становится недоступным, это означает, что клиент отключился.

Мы предоставляем подписке пару ссылок на методы, `this::onClientConnected` и `this::onClientDisconnected` , которые будут вызываться, когда _image_ становится доступным и недоступным соответственно.

[EchoServer.onClientConnected \(\)](https://github.com/io7m/aeron-guide/blob/f967e09d36ec8c7f525a66ade2f6f00ab43dcd51/src/main/java/com/io7m/aeron_guide/take1/EchoServer.java#L331)

```java
  private void onClientConnected(final Image image) {
    final int session = image.sessionId();
    LOG.debug("onClientConnected: {}", image.sourceIdentity());

    this.clients.put(
      Integer.valueOf(session),
      new ServerClient(session, image, this.aeron));
  }
```

Когда _image_ становится доступным, мы записываем _image session ID_. Это может быть эффективно использовано для уникальной идентификации клиента в отношении этой конкретной _подписки_ . Мы создаем новый экземпляр класса `ServerClient` используемый для хранения состояния каждого клиента на сервере, и сохраняем его в map, связывая _session IDs_ с экземплярами `ServerClient` . Детали класса `ServerClient` будут обсуждены в ближайшее время.

Аналогично, в методе `onClientDisconnected` мы находим клиент, который, по-видимому, отключается с помощью _image session ID_, вызываем метод `close` в соответствующем экземпляре `ServerClient` , предполагая, что он существует, и удаляем экземпляр`ServerClient` из map'ы клиентов.

[EchoServer.onClientDisconnected \(\)](https://github.com/io7m/aeron-guide/blob/f967e09d36ec8c7f525a66ade2f6f00ab43dcd51/src/main/java/com/io7m/aeron_guide/take1/EchoServer.java#L318)

```java
  private void onClientDisconnected(final Image image) {
    final int session = image.sessionId();
    LOG.debug("onClientDisconnected: {}", image.sourceIdentity());

    try (final ServerClient client = this.clients.remove(Integer.valueOf(session))) {
      LOG.debug("onClientDisconnected: closing client {}", client);
    } catch (final Exception e) {
      LOG.error("onClientDisconnected: failed to close client: ", e);
    }
  }
```

Сервер не создает _публикацию_ в методе `run` как это сделал клиент: он откладывает создание _публикаций_ до тех пор, пока клиенты не подключатся по причинам, которые будут обсуждаться в ближайшее время.

Поэтому метод полного `run` выглядит следующим образом:

[EchoServer.run \(\)](https://github.com/io7m/aeron-guide/blob/f967e09d36ec8c7f525a66ade2f6f00ab43dcd51/src/main/java/com/io7m/aeron_guide/take1/EchoServer.java#L224)

```java
  public void run() throws Exception {
    try (final Subscription sub = this.setupSubscription()) {
      this.runLoop(sub);
    }
  }
```

Метод `runLoop` на сервере упрощен по сравнению с аналогичным методом на клиенте. Метод просто опрашивает основную подписку несколько раз:

[EchoServer.runLoop \(\)](https://github.com/io7m/aeron-guide/blob/f967e09d36ec8c7f525a66ade2f6f00ab43dcd51/src/main/java/com/io7m/aeron_guide/take1/EchoServer.java#L232)

```java
  private void runLoop(final Subscription sub) throws InterruptedException {
    final FragmentHandler assembler = new FragmentAssembler(this::onParseMessage);

    while (true) {
      if (sub.isConnected()) {
        sub.poll(assembler, 10);
      }

      Thread.sleep(100L);
    }
  }
```

Основным отличием является работа, выполняемая в методе `onParseMessage` :

[EchoServer.onParseMessage \(\)](https://github.com/io7m/aeron-guide/blob/f967e09d36ec8c7f525a66ade2f6f00ab43dcd51/src/main/java/com/io7m/aeron_guide/take1/EchoServer.java#L248)

```java
  private void onParseMessage(
    final DirectBuffer buffer,
    final int offset,
    final int length,
    final Header header)
  {
    final int session = header.sessionId();

    final ServerClient client = this.clients.get(Integer.valueOf(session));
    if (client == null) {
      LOG.warn(
        "received message from unknown client: {}",
        Integer.valueOf(session));
      return;
    }

    final byte[] buf = new byte[length];
    buffer.getBytes(offset, buf);
    final String message = new String(buf, UTF_8);
    client.onReceiveMessage(message);
  }
```

Сначала мы берем _session ID,_ предоставленный нам значением `Header` переданного нам Aeron'ом. Идентификатор _сеанса_ используется для поиска клиента в map'е клиентов, заполненной методом `onClientConnected` . Предполагая, что клиент фактически существует с совпадающим _session ID_ , строка UTF-8 декодируется из буфера, как это происходит в реализации `EchoClient` , но затем декодированная строка передается соответствующему экземпляру `ServerClient` для обработки с помощью его метода `onReceiveMessage` .

Из-за небольшого размера класса `ServerClient` \(он фактически содержит только один метод, выполняющий интересную работу\), код публикуется здесь полностью:

[EchoServer.ServerClient](https://github.com/io7m/aeron-guide/blob/f967e09d36ec8c7f525a66ade2f6f00ab43dcd51/src/main/java/com/io7m/aeron_guide/take1/EchoServer.java#L63)

```java
  private static final class ServerClient implements AutoCloseable {
    private static final Pattern HELLO_PATTERN =
      Pattern.compile("HELLO ([0-9]+)");

    private enum State
    {
      INITIAL,
      CONNECTED
    }

    private final int session;
    private final Image image;
    private final Aeron aeron;
    private State state;
    private final UnsafeBuffer buffer;
    private Publication publication;

    ServerClient(
      final int session,
      final Image in_image,
      final Aeron in_aeron)
    {
      this.session = session;
      this.image = Objects.requireNonNull(in_image, "image");
      this.aeron = Objects.requireNonNull(in_aeron, "aeron");
      this.state = State.INITIAL;
      this.buffer = new UnsafeBuffer(
        BufferUtil.allocateDirectAligned(2048, 16));
    }

    @Override
    public void close() throws Exception {
      closeIfNotNull(this.publication);
    }

    public void onReceiveMessage(final String message) {
      Objects.requireNonNull(message, "message");

      LOG.debug(
        "receive [0x{}]: {}",
        Integer.toUnsignedString(this.session),
        message);

      switch (this.state) {
        case INITIAL: {
          this.onReceiveMessageInitial(message);
          break;
        }
        case CONNECTED: {
          sendMessage(this.publication, this.buffer, message);
          break;
        }
      }
    }

    private void onReceiveMessageInitial(final String message) {
      final Matcher matcher = HELLO_PATTERN.matcher(message);
      if (!matcher.matches()) {
        LOG.warn("client sent malformed HELLO message: {}", message);
        return;
      }

      final int port = Integer.parseUnsignedInt(matcher.group(1));
      final String source_id = this.image.sourceIdentity();

      try {
        final URI source_uri = new URI("fake://" + source_id);

        final String address = new StringBuilder(64)
            .append(source_uri.getHost())
            .append(":")
            .append(port)
            .toString();

        final String pub_uri = new ChannelUriStringBuilder()
            .reliable(TRUE)
            .media("udp")
            .endpoint(address)
            .build();

        this.publication = this.aeron.addPublication(pub_uri, ECHO_STREAM_ID);

        this.state = State.CONNECTED;
      } catch (final URISyntaxException e) {
        LOG.warn("client sent malformed HELLO message: {}: ", message, e);
      }
    }
  }
```

Класс `ServerClient` поддерживает поле `State` которое может быть либо `CONNECTED` либо `INITIAL` .Клиент начинает в `INITIAL` состоянии, а затем успешно переходит в состояние `CONNECTED` после успешной обработки строки `HELLO` которая, как [ожидается, будет отправлена ​​подключающимися клиентами в качестве их первого сообщения](overview.md) . Метод `onReceiveMessage` проверяет, находится ли клиент в `INITIAL` состоянии или в состоянии `CONNECTED` . Если клиент находится в состоянии `onReceiveMessageInitial` , сообщение передается методу `onReceiveMessageInitial` . Этот метод анализирует то, что предполагает, что это будет строка `HELLO` и создает новую _публикацию,_ которая будет использоваться для отправки сообщений обратно клиенту. Aeron предоставляет нам как исходный адрес клиента, так и эфемерный порт, который клиент использовал для отправки сообщения, которое мы только что получили через метод `Image.sourceIdentity()` . Однако мы не можем отправлять сообщения обратно на временный порт, который использовал клиент: нам нужно отправлять сообщения на порт, указанный клиентом в сообщении `HELLO` чтобы их можно было прочитать с помощью _подписки,_ созданной `EchoClient.setupSubscription()` для этой цели в `EchoClient.setupSubscription()` ,

При использовании UDP в качестве транспорта результатом вызова `sourceIdentity()` будет строка вида `ip-address:port` . Для клиента `10.10.1.200` использующего произвольный эфемерный UDP-порт с большим номером, строка может выглядеть примерно так: `10.10.1.200:53618` . Самый простой способ разобрать строку этой формы - просто делегировать разбор стандартному классу `java.net.URI` . Мы делаем это путем создания URI, содержащего исходный адрес и порт, извлечения IP-адреса из полученного значения `URI` , замены порта, указанного клиентом в строке `HELLO` , и последующего открытия новой _публикации_ способом, который теперь должен быть знакомый.

> Мы используем `fake://`схему URI просто потому , что `java.net.URI`класс требует _в_ схеме некоторого описания , если мы хотим, чтобы разобрать вход в `host`и `port`сегментах.

Если предположить, что все это происходит без проблем, клиент переходит в состояние `CONNECTED` и метод возвращается. С этого момента любое сообщение, полученное этим конкретным клиентским экземпляром, будет отправлено обратно клиенту через вновь созданную _публикацию_ .

На данный момент у нас, кажется, есть работающий клиент и сервер. На сервер добавлен дополнительный метод `main` для помощи в тестировании из командной строки:

[EchoServer.main \(\)](https://github.com/io7m/aeron-guide/blob/f967e09d36ec8c7f525a66ade2f6f00ab43dcd51/src/main/java/com/io7m/aeron_guide/take1/EchoServer.java#L349)

```java
  public static void main(
    final String[] args)
    throws Exception
  {
    if (args.length < 3) {
      LOG.error("usage: directory local-address local-port");
      System.exit(1);
    }

    final Path directory = Paths.get(args[0]);
    final InetAddress local_name = InetAddress.getByName(args[1]);
    final Integer local_port = Integer.valueOf(args[2]);

    final InetSocketAddress local_address =
      new InetSocketAddress(local_name, local_port.intValue());

    try (final EchoServer server = create(directory, local_address)) {
      server.run();
    }
  }
```

Выполнение сервера и клиента из командной строки приводит к ожидаемому результату:

```bash
server$ java -classpath target/com.io7m.aeron-guide-0.0.1.jar com.io7m.aeron_guide.take1.EchoServer /tmp/aeron-server 10.10.1.100 9000
20:26:47.070 [main] DEBUG com.io7m.aeron_guide.take1.EchoServer - subscription URI: aeron:udp?endpoint=10.10.1.100:9000|reliable=true
20:28:09.981 [aeron-client-conductor] DEBUG com.io7m.aeron_guide.take1.EchoServer - onClientConnected: 10.10.1.100:44501
20:28:10.988 [main] DEBUG com.io7m.aeron_guide.take1.EchoServer - receive [0x896291375]: HELLO 8000
20:28:11.049 [main] DEBUG com.io7m.aeron_guide.take1.EchoServer - receive [0x896291375]: 2745822766
20:28:11.050 [main] DEBUG com.io7m.aeron_guide.take1.EchoServer - send: [session 0x562238613] 2745822766
20:28:11.953 [main] DEBUG com.io7m.aeron_guide.take1.EchoServer - receive [0x896291375]: 1016181810
20:28:11.953 [main] DEBUG com.io7m.aeron_guide.take1.EchoServer - send: [session 0x562238613] 1016181810
20:28:12.955 [main] DEBUG com.io7m.aeron_guide.take1.EchoServer - receive [0x896291375]: 296510575
20:28:12.955 [main] DEBUG com.io7m.aeron_guide.take1.EchoServer - send: [session 0x562238613] 296510575
20:28:13.957 [main] DEBUG com.io7m.aeron_guide.take1.EchoServer - receive [0x896291375]: 3276793170
20:28:13.957 [main] DEBUG com.io7m.aeron_guide.take1.EchoServer - send: [session 0x562238613] 3276793170

client$ java -classpath target/com.io7m.aeron-guide-0.0.1.jar com.io7m.aeron_guide.take1.EchoClient /tmp/aeron-client0 10.10.1.100 8000 10.10.1.100 9000
20:28:09.826 [main] DEBUG com.io7m.aeron_guide.take1.EchoClient - subscription URI: aeron:udp?endpoint=10.10.1.100:8000|reliable=true
20:28:09.846 [main] DEBUG com.io7m.aeron_guide.take1.EchoClient - publication URI: aeron:udp?endpoint=10.10.1.100:9000|reliable=true
20:28:10.926 [main] DEBUG com.io7m.aeron_guide.take1.EchoClient - send: HELLO 8000
20:28:10.927 [main] DEBUG com.io7m.aeron_guide.take1.EchoClient - send: 2745822766
20:28:11.928 [main] DEBUG com.io7m.aeron_guide.take1.EchoClient - send: 1016181810
20:28:11.933 [main] DEBUG com.io7m.aeron_guide.take1.EchoClient - response: 2745822766
20:28:12.934 [main] DEBUG com.io7m.aeron_guide.take1.EchoClient - send: 296510575
20:28:12.934 [main] DEBUG com.io7m.aeron_guide.take1.EchoClient - response: 1016181810
20:28:13.935 [main] DEBUG com.io7m.aeron_guide.take1.EchoClient - send: 3276793170
20:28:13.935 [main] DEBUG com.io7m.aeron_guide.take1.EchoClient - response: 296510575
```

Реализация работает, но страдает от ряда недостатков, начиная от доброкачественных до потенциально фатальных.  


