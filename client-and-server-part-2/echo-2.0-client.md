# Echo 2.0 Client

Реализация клиента только немного сложнее, чем исходный `echo`клиент.

Метод `create()` для клиента имеет такие же модификации, сделанные в реализации сервера \(например, введение зарезервированного диапазона _session IDs_\):

[EchoClient.create\(\)](https://github.com/io7m/aeron-guide/blob/f967e09d36ec8c7f525a66ade2f6f00ab43dcd51/src/main/java/com/io7m/aeron_guide/take2/EchoClient.java#L79)

```java
  public static EchoClient create(
    final EchoClientConfiguration configuration)
    throws EchoClientException
  {
    Objects.requireNonNull(configuration, "configuration");

    final String directory =
      configuration.baseDirectory()
        .toAbsolutePath()
        .toString();

    final MediaDriver.Context media_context =
      new MediaDriver.Context()
        .dirDeleteOnStart(true)
        .publicationReservedSessionIdLow(EchoSessions.RESERVED_SESSION_ID_LOW)
        .publicationReservedSessionIdHigh(EchoSessions.RESERVED_SESSION_ID_HIGH)
        .aeronDirectoryName(directory);

    final Aeron.Context aeron_context =
      new Aeron.Context().aeronDirectoryName(directory);

    MediaDriver media_driver = null;

    try {
      media_driver = MediaDriver.launch(media_context);

      Aeron aeron = null;
      try {
        aeron = Aeron.connect(aeron_context);
      } catch (final Exception e) {
        closeIfNotNull(aeron);
        throw e;
      }

      return new EchoClient(media_driver, aeron, configuration);
    } catch (final Exception e) {
      try {
        closeIfNotNull(media_driver);
      } catch (final Exception c_ex) {
        e.addSuppressed(c_ex);
      }
      throw new EchoClientCreationException(e);
    }
  }
```

Метод `run()` несколько сложнее:

[EchoClient.run\(\)](https://github.com/io7m/aeron-guide/blob/f967e09d36ec8c7f525a66ade2f6f00ab43dcd51/src/main/java/com/io7m/aeron_guide/take2/EchoClient.java#L175)

```java
  public void run()
    throws EchoClientException
  {
    /*
     * Generate a one-time pad.
     */

    this.duologue_key = this.random.nextInt();

    final UnsafeBuffer buffer =
      new UnsafeBuffer(BufferUtil.allocateDirectAligned(1024, 16));

    final String session_name;
    try (final Subscription subscription = this.setupAllClientsSubscription()) {
      try (final Publication publication = this.setupAllClientsPublication()) {

        /*
         * Send a one-time pad to the server.
         */

        EchoMessages.sendMessage(
          publication,
          buffer,
          "HELLO " + Integer.toUnsignedString(this.duologue_key, 16).toUpperCase());

        session_name = Integer.toString(publication.sessionId());
        this.waitForConnectResponse(subscription, session_name);
      } catch (final IOException e) {
        throw new EchoClientIOException(e);
      }
    }

    /*
     * Connect to the publication and subscription that the server has sent
     * back to this client.
     */

    try (final Subscription subscription = this.setupConnectSubscription()) {
      try (final Publication publication = this.setupConnectPublication()) {
        this.runEchoLoop(buffer, session_name, subscription, publication);
      } catch (final IOException e) {
        throw new EchoClientIOException(e);
      }
    }
  }
```

Код должен быть достаточно понятным. Мы настраиваем _публикацию_ и _подписку_ на канал _всех клиентов_ сервера и отправляем `HELLO`сообщение с помощью одноразовой панели, с которой мы ожидаем, что сервер зашифрует ответ. На каждом этапе мы ожидаем _подключения публикаций_ и _подписок_ , или мы делаем тайм-аут и громко терпим неудачу с ошибкой, если они этого не делают \(решая [проблемы, отмеченные при отключениях](../client-and-server-part-1/implementation-weaknesses.md#otklyuchenie-servera-echoclient) \).

Предполагая, что мы получаем все данные, которые мы ожидаем от сервера, мы входим в теперь знакомый `runEchoLoop`метод:

[EchoClient.runEchoLoop\(\)](https://github.com/io7m/aeron-guide/blob/f967e09d36ec8c7f525a66ade2f6f00ab43dcd51/src/main/java/com/io7m/aeron_guide/take2/EchoClient.java#L221)

```java
  private void runEchoLoop(
    final UnsafeBuffer buffer,
    final String session_name,
    final Subscription subscription,
    final Publication publication)
    throws IOException
  {
    final FragmentHandler handler =
      new FragmentAssembler(
        (data, offset, length, header) ->
          onEchoResponse(session_name, data, offset, length));

    while (true) {

      /*
       * Send ECHO messages to the server and wait for responses.
       */

      EchoMessages.sendMessage(
        publication,
        buffer,
        "ECHO " + Long.toUnsignedString(this.random.nextLong(), 16));

      for (int index = 0; index < 100; ++index) {
        subscription.poll(handler, 1000);

        try {
          Thread.sleep(10L);
        } catch (final InterruptedException e) {
          Thread.currentThread().interrupt();
        }
      }
    }
  }
```

Использование `EchoMessages.sendMessage()`приведет к разрыву цикла с исключением, если сервер отключит клиента.

