# Echo Клиент

Мы начнем с определения класса `EchoClient` с помощью статического метода `create` который инициализирует media driver и экземпляр библиотеки Aeron. Для медиа-драйвера требуется каталог в файловой системе, в котором он создает различные временные файлы, отображаемые в память, чтобы обеспечить эффективную поточно-ориентированную связь между отдельными компонентами библиотеки. Для достижения наилучших результатов этот каталог должен находиться в файловой системе с поддержкой памяти \(например, `/dev/shm` в Linux\), но на самом деле это не требуется.

Метод `create` который мы определяем, просто создает и запускает Aeron и медиа-драйвер. Он также принимает локальный адрес, который клиент будет использовать для связи, и адрес сервера, к которому клиент будет подключаться. Он передает эти адреса конструктору для последующего использования.

[EchoClient.create \(\)](https://github.com/io7m/aeron-guide/blob/f967e09d36ec8c7f525a66ade2f6f00ab43dcd51/src/main/java/com/io7m/aeron_guide/take1/EchoClient.java#L71)

```java
  public static EchoClient create(
    final Path media_directory,
    final InetSocketAddress local_address,
    final InetSocketAddress remote_address)
    throws Exception
  {
    Objects.requireNonNull(media_directory, "media_directory");
    Objects.requireNonNull(local_address, "local_address");
    Objects.requireNonNull(remote_address, "remote_address");

    final String directory =
      media_directory.toAbsolutePath().toString();

    final MediaDriver.Context media_context =
      new MediaDriver.Context()
        .dirDeleteOnStart(true)
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

      return new EchoClient(media_driver, aeron, local_address, remote_address);
    } catch (final Exception e) {
      closeIfNotNull(media_driver);
      throw e;
    }
  }
```

Различные типы `Context` содержат множество параметров конфигурации. Ради этого примера мы заинтересованы только в настройке расположения каталогов и в противном случае будем использовать настройки по умолчанию.

Поскольку класс `EchoClient` отвечает за запуск драйвера мультимедиа, он также отвечает за выключение драйвера мультимедиа и Aeron при необходимости. Он реализует стандартный интерфейс`java.io.Closeable` и выполняет необходимую очистку в методе `close` .

[EchoClient.close \(\)](https://github.com/io7m/aeron-guide/blob/f967e09d36ec8c7f525a66ade2f6f00ab43dcd51/src/main/java/com/io7m/aeron_guide/take1/EchoClient.java#L239)

```java
 public void close()
  {
    this.aeron.close();
    this.media_driver.close();
  }
```

Теперь нам нужно определить простой \(блокирующий\) метод `run` который пытается подключиться к серверу, а затем переходит в бесконечный цикл отправки и получения сообщений. Для получения удобочитаемого вывода мы ограничимся опросом сообщений один раз в секунду. В реальных приложениях с требованиями к малой задержке это, очевидно, будет абсолютно контрпродуктивным, и будет использоваться более ощутимая задержка. Метод состоит из нескольких частей, поэтому мы определим каждую из них как свои собственные методы, а затем составим их в конце.

> Интерфейс `Subscription` также содержит много различных вариаций `poll`которые обеспечивают различные семантики в зависимости от требований приложения.

Во-первых, нам нужно создать _подписку,_ которая будет использоваться для получения сообщений с удаленного сервера. Для тех, кто знаком с программированием UDP, это несколько аналогично открытию сокета UDP и привязке его к локальному адресу и порту. Мы создаем подписку, создавая [URI канала](https://github.com/real-logic/aeron/wiki/Channel-Configuration) на основе локального адреса, указанного в методе `create` . Мы используем удобный класс `ChannelUriStringBuilder` для создания строки URI, указывающей детали подписки. Мы заявляем, что хотим использовать `udp` в качестве основного транспорта и хотим, чтобы канал был _надежным_ \(это означает, что потерянные сообщения будут повторно переданы, а сообщения будут доставлены на удаленную сторону в том порядке, в котором они были отправлены\). Мы также указываем [stream ID](https://github.com/real-logic/aeron/wiki/Protocol-Specification#stream-setup) при создании подписки. Aeron способен мультиплексировать несколько независимых _потоков_ сообщений в одно соединение. Поэтому клиенту и серверу необходимо согласовать идентификатор потока, который будет использоваться для связи. В этом случае мы просто выбираем произвольное значение `0x2044f002` , но можно использовать любое ненулевое 32-разрядное значение без знака; Выбор полностью за приложением.

[EchoClient.setupSubscription \(\)](https://github.com/io7m/aeron-guide/blob/f967e09d36ec8c7f525a66ade2f6f00ab43dcd51/src/main/java/com/io7m/aeron_guide/take1/EchoClient.java#L226)

```java
  private Subscription setupSubscription()
  {
    final String sub_uri =
      new ChannelUriStringBuilder()
        .reliable(TRUE)
        .media("udp")
        .endpoint(this.local_address.toString().replaceFirst("^/", ""))
        .build();

    LOG.debug("subscription URI: {}", sub_uri);
    return this.aeron.addSubscription(sub_uri, ECHO_STREAM_ID);
  }
```

Если для этого примера мы предполагаем, что клиент на `10.10.1.100` использует локальный порт `8000` , результирующий URI канала будет выглядеть примерно так:

```text
 aeron:udp?endpoint=10.10.1.100:8000|reliable=true 
```

Затем мы создаем _публикацию,_ которая будет использоваться для отправки сообщений на сервер.Процедура создания публикации очень похожа на процедуру _подписки_ , поэтому объяснение здесь повторяться не будет.

[EchoClient.setupPublication \(\)](https://github.com/io7m/aeron-guide/blob/f967e09d36ec8c7f525a66ade2f6f00ab43dcd51/src/main/java/com/io7m/aeron_guide/take1/EchoClient.java#L213)

```java
  private Publication setupPublication()
  {
    final String pub_uri =
      new ChannelUriStringBuilder()
        .reliable(TRUE)
        .media("udp")
        .endpoint(this.remote_address.toString().replaceFirst("^/", ""))
        .build();

    LOG.debug("publication URI: {}", pub_uri);
    return this.aeron.addPublication(pub_uri, ECHO_STREAM_ID);
  }
```

Если в этом примере мы предположим, что сервер `10.10.1.200` использует локальный порт `9000` , результирующий URI канала будет выглядеть примерно так:

```text
 aeron:udp?endpoint=10.10.1.200:9000|reliable=true 
```

Теперь мы определим метод `runLoop` который принимает созданную публикацию и подписку и просто зацикливается навсегда, отправляя и получая сообщения.

[EchoClient.runLoop \(\)](https://github.com/io7m/aeron-guide/blob/f967e09d36ec8c7f525a66ade2f6f00ab43dcd51/src/main/java/com/io7m/aeron_guide/take1/EchoClient.java#L131)

```java
  private void runLoop(
    final Subscription sub,
    final Publication pub)
    throws InterruptedException
  {
    final UnsafeBuffer buffer =
      new UnsafeBuffer(BufferUtil.allocateDirectAligned(2048, 16));

    final Random random = new Random();

    /*
     * Try repeatedly to send an initial HELLO message
     */

    while (true) {
      if (pub.isConnected()) {
        if (sendMessage(pub, buffer, "HELLO " + this.local_address.getPort())) {
          break;
        }
      }

      Thread.sleep(1000L);
    }

    /*
     * Send an infinite stream of random unsigned integers.
     */

    final FragmentHandler assembler =
      new FragmentAssembler(EchoClient::onParseMessage);

    while (true) {
      if (pub.isConnected()) {
        sendMessage(pub, buffer, Integer.toUnsignedString(random.nextInt()));
      }
      if (sub.isConnected()) {
        sub.poll(assembler, 10);
      }
      Thread.sleep(1000L);
    }
  }
```

Сначала метод создает буфер, который будет использоваться для хранения входящих и исходящих данных. Некоторые из способов, которыми Aeron достигает очень высокой степени производительности, включают использование встроенной памяти, выделение памяти заранее, чтобы избежать образования мусора во время стационарного выполнения, и устранение максимально возможного копирования в буфер на JVM. Чтобы помочь в этом, мы выделяем 2- `2KiB` прямой байтовый буфер \(с `16` байтовым выравниванием\) для хранения сообщений и используем класс `UnsafeBuffer` из пакета Aeron, связанного со структурой данных [Agrona](https://github.com/real-logic/Agrona)[,](https://translate.googleusercontent.com/translate_c?depth=1&rurl=translate.google.com&sl=en&sp=nmt4&tl=ru&u=https://github.com/real-logic/Agrona&xid=17259,1500004,15700023,15700186,15700190,15700253,15700256,15700259&usg=ALkJrhgOB1TbN23b4isB6KLOfl6eEONOeQ) для получения очень высокопроизводительных \(небезопасных\) операций с памятью на заданном буфере.

Затем метод отправляет строку `HELLO <port>` где `<port>` заменяется номером порта, использованным для создания _подписки_ ранее. Сервер должен адресовать ответы на этот порт, поэтому они будут доступны в _подписке,_ которую открыл клиент.

Затем метод зацикливается навсегда, опрашивая подписку на новые сообщения и отправляя в публикацию случайную целочисленную строку, ожидая секунду на каждой итерации.

Метод `sendMessage` - это более или менее неинтересный служебный метод, который просто упаковывает данную строку в буфер, который мы выделили в начале метода. Он не выполняет какой-либо конкретной обработки ошибок: отправка сообщения может завершиться неудачей по причинам, указанным в документации по методу `Publication.offer()` , и реальные приложения должны выполнить соответствующую обработку ошибок. Метод также не пытается проверить, не переполнит ли содержимое строки буфер. Это может иметь впечатляющие последствия, учитывая природу `Unsafe`буферов. Наша реализация просто пытается написать сообщение, повторяя до пяти раз, прежде чем сдаться. Метод возвращает `true` если сообщение было отправлено, регистрирует сообщение об ошибке и возвращает `false` противном случае. Лучшие подходы для реальных приложений [обсуждаются позже](implementation-weaknesses.md#otpravka-soobshenii-ne-yavlyaetsya-nadezhnoi) .

[EchoClient.sendMessage \(\)](https://github.com/io7m/aeron-guide/blob/f967e09d36ec8c7f525a66ade2f6f00ab43dcd51/src/main/java/com/io7m/aeron_guide/take1/EchoClient.java#L185)

```java
  private static boolean sendMessage(
    final Publication pub,
    final UnsafeBuffer buffer,
    final String text)
  {
    LOG.debug("send: {}", text);

    final byte[] value = text.getBytes(UTF_8);
    buffer.putBytes(0, value);

    long result = 0L;
    for (int index = 0; index < 5; ++index) {
      result = pub.offer(buffer, 0, text.length());
      if (result < 0L) {
        try {
          Thread.sleep(100L);
        } catch (final InterruptedException e) {
          Thread.currentThread().interrupt();
        }
        continue;
      }
      return true;
    }

    LOG.error("could not send: {}", Long.valueOf(result));
    return false;
  }
```

Метод `poll` , определенный для типа `Subscription` , принимает в качестве аргумента функцию типа `FragmentHandler` . В нашем случае мы передаем реализованную Aeron реализацию типа `FragmentHandler` называется `FragmentAssembler` . Класс `FragmentAssembler` обрабатывает повторную сборку фрагментированных сообщений, а затем передает собранные сообщения в наш собственный `FragmentHandler` \(в данном случае, метод `onParseMessage` \). Наш код вряд ли когда-либо отправит или получит сообщение, которое могло бы превысить [MTU](https://ru.wikipedia.org/wiki/Maximum_transmission_unit) базового транспорта UDP. Если это произойдет, `FragmentAssembler` позаботится об этом.

[EchoClient.onParseMessage \(\)](https://github.com/io7m/aeron-guide/blob/f967e09d36ec8c7f525a66ade2f6f00ab43dcd51/src/main/java/com/io7m/aeron_guide/take1/EchoClient.java#L173)

```java
  private static void onParseMessage(
    final DirectBuffer buffer,
    final int offset,
    final int length,
    final Header header)
  {
    final byte[] buf = new byte[length];
    buffer.getBytes(offset, buf);
    final String response = new String(buf, UTF_8);
    LOG.debug("response: {}", response);
  }  
```

Теперь, когда у нас есть все необходимые части, метод `run` тривиален:

[EchoClient.run \(\)](https://github.com/io7m/aeron-guide/blob/f967e09d36ec8c7f525a66ade2f6f00ab43dcd51/src/main/java/com/io7m/aeron_guide/take1/EchoClient.java#L121)

```java
  public void run()
    throws Exception
  {
    try (final Subscription sub = this.setupSubscription()) {
      try (final Publication pub = this.setupPublication()) {
        this.runLoop(sub, pub);
      }
    }
  }   
```

Это тот минимум, который необходим для работы клиента. Для удобства тестирования можно определить простой метод `main` который принимает аргументы командной строки:

[EchoClient.main \(\)](https://github.com/io7m/aeron-guide/blob/f967e09d36ec8c7f525a66ade2f6f00ab43dcd51/src/main/java/com/io7m/aeron_guide/take1/EchoClient.java#L246)

```java
  public static void main(
    final String[] args)
    throws Exception
  {
    if (args.length < 5) {
      LOG.error("usage: directory local-address local-port remote-address remote-port");
      System.exit(1);
    }

    final Path directory = Paths.get(args[0]);
    final InetAddress local_name = InetAddress.getByName(args[1]);
    final Integer local_port = Integer.valueOf(args[2]);
    final InetAddress remote_name = InetAddress.getByName(args[3]);
    final Integer remote_port = Integer.valueOf(args[4]);

    final InetSocketAddress local_address =
      new InetSocketAddress(local_name, local_port.intValue());
    final InetSocketAddress remote_address =
      new InetSocketAddress(remote_name, remote_port.intValue());

    try (final EchoClient client = create(directory, local_address, remote_address)) {
      client.run();
    }
  }
```

