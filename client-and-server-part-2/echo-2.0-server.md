# Echo 2.0 Сервер

Мы начнем с изучения реализации сервера.

Метод `create` в основном без изменений. Основное отличие заключается в том , что мы говорим _media driver_, что мы собираемся вручную назначить _идентификаторы сеансов_ в диапазоне `[EchoSessions.RESERVED_SESSION_ID_LOW, EchoSessions.RESERVED_SESSION_ID_HIGH]`самого. Когда _медиа-драйвер_ автоматически назначает _session IDs_ , он должен использовать значения вне этого диапазона, чтобы избежать конфликта с любым, который мы назначаем сами. Мы также инициализируем значение типа `EchoServerExecutorService`.

[EchoServer.create\(\)](https://github.com/io7m/aeron-guide/blob/f967e09d36ec8c7f525a66ade2f6f00ab43dcd51/src/main/java/com/io7m/aeron_guide/take2/EchoServer.java#L94)

```java
  public static EchoServer create(
    final Clock clock,
    final EchoServerConfiguration configuration)
    throws EchoServerException
  {
    Objects.requireNonNull(clock, "clock");
    Objects.requireNonNull(configuration, "configuration");

    final String directory =
      configuration.baseDirectory().toAbsolutePath().toString();

    final MediaDriver.Context media_context =
      new MediaDriver.Context()
        .dirDeleteOnStart(true)
        .publicationReservedSessionIdLow(EchoSessions.RESERVED_SESSION_ID_LOW)
        .publicationReservedSessionIdHigh(EchoSessions.RESERVED_SESSION_ID_HIGH)
        .aeronDirectoryName(directory);

    final Aeron.Context aeron_context =
      new Aeron.Context()
        .aeronDirectoryName(directory);

    EchoServerExecutorService exec = null;
    try {
      exec = EchoServerExecutor.create();

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

        return new EchoServer(clock, exec, media_driver, aeron, configuration);
      } catch (final Exception e) {
        closeIfNotNull(media_driver);
        throw e;
      }
    } catch (final Exception e) {
      try {
        closeIfNotNull(exec);
      } catch (final Exception c_ex) {
        e.addSuppressed(c_ex);
      }
      throw new EchoServerCreationException(e);
    }
  }
```

Как можно видеть, несколько параметров, которые были переданы `create()`методу в исходной реализации, были перемещены в неизменяемый `EchoServerConfiguration`тип. Реализация этого типа генерируется автоматически процессором аннотаций [immutables.org](http://www.immutables.org/).

[EchoServerConfiguration](https://github.com/io7m/aeron-guide/blob/f967e09d36ec8c7f525a66ade2f6f00ab43dcd51/src/main/java/com/io7m/aeron_guide/take2/EchoServerConfiguration.java#L9)

```java
public interface EchoServerConfiguration
{
  /**
   * @return The base directory that will be used for the server; should be unique for each server instance
   */

  @Value.Parameter
  Path baseDirectory();

  /**
   * @return The local address to which the server will bind
   */

  @Value.Parameter
  InetAddress localAddress();

  /**
   * @return The port that the server will use for client introductions
   */

  @Value.Parameter
  int localInitialPort();

  /**
   * @return The dynamic MDC control port that the server will use for client introductions
   */

  @Value.Parameter
  int localInitialControlPort();

  /**
   * @return The base port that will be used for individual client duologues
   */

  @Value.Parameter
  int localClientsBasePort();

  /**
   * @return The maximum number of clients that will be allowed on the server
   */

  @Value.Parameter
  int clientMaximumCount();

  /**
   * @return The maximum number of duologues that will be allowed per remote client IP address
   */

  @Value.Parameter
  int maximumConnectionsPerAddress();
}
```

`EchoServerExecutorService`Тип эффективен однопоточный `java.util.concurrent.Executor`с некоторыми дополнительными методами добавил , что код можно использовать для проверки , что это действительно работает на правильном потоке. Этот класс используется для того, чтобы гарантировать, что длительная работа не выполняется в потоках, принадлежащих Aeron \(устранение [отмеченных проблем](../client-and-server-part-1/implementation-weaknesses.md#rabota-na-nityakh-aeron) с потоками \), а также для обеспечения того, чтобы доступ к внутреннему состоянию сервера был строго однопоточным. Реализация этого класса довольно неудивительна:

[EchoServerExecutorService](https://github.com/io7m/aeron-guide/blob/f967e09d36ec8c7f525a66ade2f6f00ab43dcd51/src/main/java/com/io7m/aeron_guide/take2/EchoServerExecutorService.java#L10)

```java
public interface EchoServerExecutorService extends AutoCloseable, Executor
{
  /**
   * @return {@code true} if the caller of this method is running on the executor thread
   */

  boolean isExecutorThread();

  /**
   * Raise {@link IllegalStateException} iff {@link #isExecutorThread()} would
   * currently return {@code false}.
   */

  default void assertIsExecutorThread()
  {
    if (!this.isExecutorThread()) {
      throw new IllegalStateException(
        "The current thread is not a server executor thread");
    }
  }
}
```

[EchoServerExecutor](https://github.com/io7m/aeron-guide/blob/f967e09d36ec8c7f525a66ade2f6f00ab43dcd51/src/main/java/com/io7m/aeron_guide/take2/EchoServerExecutor.java#L15)

```java
public final class EchoServerExecutor implements EchoServerExecutorService
{
  private static final Logger LOG =
    LoggerFactory.getLogger(EchoServerExecutor.class);

  private final ExecutorService executor;

  private EchoServerExecutor(
    final ExecutorService in_exec)
  {
    this.executor = Objects.requireNonNull(in_exec, "exec");
  }

  @Override
  public boolean isExecutorThread()
  {
    return Thread.currentThread() instanceof EchoServerThread;
  }

  @Override
  public void execute(final Runnable runnable)
  {
    Objects.requireNonNull(runnable, "runnable");

    this.executor.submit(() -> {
      try {
        runnable.run();
      } catch (final Throwable e) {
        LOG.error("uncaught exception: ", e);
      }
    });
  }

  @Override
  public void close()
  {
    this.executor.shutdown();
  }

  private static final class EchoServerThread extends Thread
  {
    EchoServerThread(final Runnable target)
    {
      super(Objects.requireNonNull(target, "target"));
    }
  }

  /**
   * @return A new executor
   */

  public static EchoServerExecutor create()
  {
    final ThreadFactory factory = r -> {
      final EchoServerThread t = new EchoServerThread(r);
      t.setName(new StringBuilder(64)
                  .append("com.io7m.aeron_guide.take2.server[")
                  .append(Long.toUnsignedString(t.getId()))
                  .append("]")
                  .toString());
      return t;
    };

    return new EchoServerExecutor(Executors.newSingleThreadExecutor(factory));
  }
}
```

Мы засоряем код вызовами, `assertIsExecutorThread()`чтобы помочь выявить неправильное использование потоков на ранних этапах.

Большая часть интересной работы, выполняемой сервером, теперь выполняется статическим внутренним классом `ClientState`. Этот класс отвечает за прием запросов от клиентов, проверку ограничений доступа \(например, принудительное ограничение ограничений для _дуологий_ по одному IP-адресу\), опрос существующих _дуологий на_ предмет активности и т. Д. Мы устанавливаем правило, согласно которому доступ к `ClientState`классу ограничен одним потоком через `EchoServerExecutor`тип. `EchoServer`Класс определяет три метода , которые каждый по существу делегируют к `ClientState`классу:

[EchoServer.onInitialClientConnected\(\)](https://github.com/io7m/aeron-guide/blob/f967e09d36ec8c7f525a66ade2f6f00ab43dcd51/src/main/java/com/io7m/aeron_guide/take2/EchoServer.java#L316)

```java
  private void onInitialClientConnected(
    final Image image)
  {
    this.executor.execute(() -> {
      LOG.debug(
        "[{}] initial client connected ({})",
        Integer.toString(image.sessionId()),
        image.sourceIdentity());

      this.clients.onInitialClientConnected(
        image.sessionId(),
        EchoAddresses.extractAddress(image.sourceIdentity()));
    });
  }
```

[EchoServer.onInitialClientDisconnected\(\)](https://github.com/io7m/aeron-guide/blob/f967e09d36ec8c7f525a66ade2f6f00ab43dcd51/src/main/java/com/io7m/aeron_guide/take2/EchoServer.java#L331)

```java
  private void onInitialClientDisconnected(
    final Image image)
  {
    this.executor.execute(() -> {
      LOG.debug(
        "[{}] initial client disconnected ({})",
        Integer.toString(image.sessionId()),
        image.sourceIdentity());

      this.clients.onInitialClientDisconnected(image.sessionId());
    });
  }
```

[EchoServer.onInitialClientMessage\(\)](https://github.com/io7m/aeron-guide/blob/f967e09d36ec8c7f525a66ade2f6f00ab43dcd51/src/main/java/com/io7m/aeron_guide/take2/EchoServer.java#L260)

```java
  private void onInitialClientMessage(
    final Publication publication,
    final DirectBuffer buffer,
    final int offset,
    final int length,
    final Header header)
  {
    final String message =
      EchoMessages.parseMessageUTF8(buffer, offset, length);

    final String session_name =
      Integer.toString(header.sessionId());
    final Integer session_boxed =
      Integer.valueOf(header.sessionId());

    this.executor.execute(() -> {
      try {
        this.clients.onInitialClientMessageProcess(
          publication,
          session_name,
          session_boxed,
          message);
      } catch (final Exception e) {
        LOG.error("could not process client message: ", e);
      }
    });
  }
```

`onInitialClientConnected()`И `onInitialClientDisconnected()`методы называются _обработчиками изображений_ по _подписке_ сервер конфигурирует для _всех клиентов_ канала. `onInitialClientMessage`Метод вызывается , когда клиент посылает сообщение на _все клиенты_канала.

`ClientState.onInitialClientConnectedProcess()`Метод делает большую часть работы , необходимой , когда клиент запрашивает создание новой _дилогии_ :

[EchoServer.ClientState.onInitialClientConnectedProcess\(\)](https://github.com/io7m/aeron-guide/blob/f967e09d36ec8c7f525a66ade2f6f00ab43dcd51/src/main/java/com/io7m/aeron_guide/take2/EchoServer.java#L400)

```java
    void onInitialClientMessageProcess(
      final Publication publication,
      final String session_name,
      final Integer session_boxed,
      final String message)
      throws EchoServerException, IOException
    {
      this.exec.assertIsExecutorThread();

      LOG.debug("[{}] received: {}", session_name, message);

      /*
       * The HELLO command is the only acceptable message from clients
       * on the all-clients channel.
       */

      final Matcher hello_matcher = PATTERN_HELLO.matcher(message);
      if (!hello_matcher.matches()) {
        EchoMessages.sendMessage(
          publication,
          this.send_buffer,
          errorMessage(session_name, "bad message"));
        return;
      }

      /*
       * Check to see if there are already too many clients connected.
       */

      if (this.client_duologues.size() >= this.configuration.clientMaximumCount()) {
        LOG.debug("server is full");
        EchoMessages.sendMessage(
          publication,
          this.send_buffer,
          errorMessage(session_name, "server full"));
        return;
      }

      /*
       * Check to see if this IP address already has the maximum number of
       * duologues allocated to it.
       */

      final InetAddress owner =
        this.client_session_addresses.get(session_boxed);

      if (this.address_counter.countFor(owner) >=
        this.configuration.maximumConnectionsPerAddress()) {
        LOG.debug("too many connections for IP address");
        EchoMessages.sendMessage(
          publication,
          this.send_buffer,
          errorMessage(session_name, "too many connections for IP address"));
        return;
      }

      /*
       * Parse the one-time pad with which the client wants the server to
       * encrypt the identifier of the session that will be created.
       */

      final int duologue_key =
        Integer.parseUnsignedInt(hello_matcher.group(1), 16);

      /*
       * Allocate a new duologue, encrypt the resulting session ID, and send
       * a message to the client telling it where to find the new duologue.
       */

      final EchoServerDuologue duologue =
        this.allocateNewDuologue(session_name, session_boxed, owner);

      final String session_crypt =
        Integer.toUnsignedString(duologue_key ^ duologue.session(), 16)
          .toUpperCase();

      EchoMessages.sendMessage(
        publication,
        this.send_buffer,
        connectMessage(
          session_name,
          duologue.portData(),
          duologue.portControl(),
          session_crypt));
    }
```

`ClientState.allocateNewDuologue()`Метод делает фактическую работу _дилогии_ распределения путем выделения номеров портов UDP, а _идентификатор сеанса_ , увеличивая количество _duologues_назначенного IP - адрес клиента, а затем вызова `EchoServerDuologue.create()`\(который делает фактическую работу по созданию основополагающего Aeron _публикаций_ и _подписок_ :

[EchoServer.ClientState.allocateNewDuologue\(\)](https://github.com/io7m/aeron-guide/blob/f967e09d36ec8c7f525a66ade2f6f00ab43dcd51/src/main/java/com/io7m/aeron_guide/take2/EchoServer.java#L486)

```java
    private EchoServerDuologue allocateNewDuologue(
      final String session_name,
      final Integer session_boxed,
      final InetAddress owner)
      throws
      EchoServerPortAllocationException,
      EchoServerSessionAllocationException
    {
      this.address_counter.increment(owner);

      final EchoServerDuologue duologue;
      try {
        final int[] ports = this.port_allocator.allocate(2);
        try {
          final int session = this.session_allocator.allocate();
          try {
            duologue =
              EchoServerDuologue.create(
                this.aeron,
                this.clock,
                this.exec,
                this.configuration.localAddress(),
                owner,
                session,
                ports[0],
                ports[1]);
            LOG.debug("[{}] created new duologue", session_name);
            this.client_duologues.put(session_boxed, duologue);
          } catch (final Exception e) {
            this.session_allocator.free(session);
            throw e;
          }
        } catch (final EchoServerSessionAllocationException e) {
          this.port_allocator.free(ports[0]);
          this.port_allocator.free(ports[1]);
          throw e;
        }
      } catch (final EchoServerPortAllocationException e) {
        this.address_counter.decrement(owner);
        throw e;
      }
      return duologue;
    }
```

Методы `ClientState.onInitialClientConnected()` и `ClientState.onInitialClientDisconnected()` просто записывают IP-адрес , связанный с каждым _session ID_ для последующего использования в методе `ClientState.onInitialClientConnectedProcess()`:

[EchoServer.ClientState.onInitialClientConnected\(\)](https://github.com/io7m/aeron-guide/blob/f967e09d36ec8c7f525a66ade2f6f00ab43dcd51/src/main/java/com/io7m/aeron_guide/take2/EchoServer.java#L538)

```java
    void onInitialClientConnected(
      final int session_id,
      final InetAddress client_address)
    {
      this.exec.assertIsExecutorThread();

      this.client_session_addresses.put(
        Integer.valueOf(session_id), client_address);
    }
```

[EchoServer.ClientState.onInitialClientDisconnected\(\)](https://github.com/io7m/aeron-guide/blob/f967e09d36ec8c7f525a66ade2f6f00ab43dcd51/src/main/java/com/io7m/aeron_guide/take2/EchoServer.java#L530)

```java
    void onInitialClientDisconnected(
      final int session_id)
    {
      this.exec.assertIsExecutorThread();

      this.client_session_addresses.remove(Integer.valueOf(session_id));
    }
```

Распределение номеров портов UDP для новых _дуологов_ осуществляется `EchoServerPortAllocator`классом. Это простой класс, который будет случайным образом распределять целочисленные значения из настраиваемого диапазона и не будет повторно использовать целочисленные значения, пока они не будут явно освобождены. В нашей реализации нам требуется 2 UDP-порта для каждого клиента, настраиваемый лимит `c`клиентов и настраиваемый базовый порт `p`. Поэтому мы устанавливаем диапазон портов на `[p, p + (2 * c))`.

[EchoServerPortAllocator](https://github.com/io7m/aeron-guide/blob/f967e09d36ec8c7f525a66ade2f6f00ab43dcd51/src/main/java/com/io7m/aeron_guide/take2/EchoServerPortAllocator.java#L17)

```java
public final class EchoServerPortAllocator
{
  private final int port_lo;
  private final int port_hi;
  private final IntHashSet ports_used;
  private final IntArrayList ports_free;

  /**
   * Create a new port allocator.
   *
   * @param port_base The base port
   * @param max_ports The maximum number of ports that will be allocated
   *
   * @return A new port allocator
   */

  public static EchoServerPortAllocator create(
    final int port_base,
    final int max_ports)
  {
    return new EchoServerPortAllocator(port_base, max_ports);
  }

  private EchoServerPortAllocator(
    final int in_port_lo,
    final int in_max_ports)
  {
    if (in_port_lo <= 0 || in_port_lo >= 65536) {
      throw new IllegalArgumentException(
        String.format(
          "Base port %d must be in the range [1, 65535]",
          Integer.valueOf(in_port_lo)));
    }

    this.port_lo = in_port_lo;
    this.port_hi = in_port_lo + (in_max_ports - 1);

    if (this.port_hi < 0 || this.port_hi >= 65536) {
      throw new IllegalArgumentException(
        String.format(
          "Uppermost port %d must be in the range [1, 65535]",
          Integer.valueOf(this.port_hi)));
    }

    this.ports_used = new IntHashSet(in_max_ports);
    this.ports_free = new IntArrayList();

    for (int port = in_port_lo; port <= this.port_hi; ++port) {
      this.ports_free.addInt(port);
    }
    Collections.shuffle(this.ports_free);
  }

  /**
   * Free a given port. Has no effect if the given port is outside of the range
   * considered by the allocator.
   *
   * @param port The port
   */

  public void free(
    final int port)
  {
    if (port >= this.port_lo && port <= this.port_hi) {
      this.ports_used.remove(port);
      this.ports_free.addInt(port);
    }
  }

  /**
   * Allocate {@code count} ports.
   *
   * @param count The number of ports that will be allocated
   *
   * @return An array of allocated ports
   *
   * @throws EchoServerPortAllocationException If there are fewer than {@code count} ports available to allocate
   */

  public int[] allocate(
    final int count)
    throws EchoServerPortAllocationException
  {
    if (this.ports_free.size() < count) {
      throw new EchoServerPortAllocationException(
        String.format(
          "Too few ports available to allocate %d ports",
          Integer.valueOf(count)));
    }

    final int[] result = new int[count];
    for (int index = 0; index < count; ++index) {
      result[index] = this.ports_free.remove(0).intValue();
      this.ports_used.add(result[index]);
    }

    return result;
  }
}
```

Точно так же распределение _идентификаторов сеансов_ для новых _дуологов_ выполняется `EchoServerSessionAllocator`классом:

> В то время как появление сделать по существу ту же работу, то `EchoServerPortAllocator`и `EchoServerSessionAllocator`классы используют разные алгоритмы из - за различий в ожидаемом диапазоне значений , которые будут выделены. `EchoServerPortAllocator`Класс ожидает , чтобы выбрать номера в небольшом диапазоне , и , следовательно , сохраняет весь диапазон памяти: Он использует `O(n)`память для диапазона `n`значений, но гарантировал `O(1)`временную сложность как для выделения и освобождения памяти. Предполагается, `EchoServerSessionAllocator`что число выбирается из диапазона не менее, `2 ^ 31`и поэтому не хранит список всех неиспользованных номеров в памяти. Он использует только `O(m)`хранилище, где `m`есть количество выделенных значений, но имеет `O(n)`сложность времени наихудшего случая для распределения, когда приближается число выделенных значений `n`.

[EchoServerSessionAllocator](https://github.com/io7m/aeron-guide/blob/f967e09d36ec8c7f525a66ade2f6f00ab43dcd51/src/main/java/com/io7m/aeron_guide/take2/EchoServerSessionAllocator.java#L23)

```java
public final class EchoServerSessionAllocator
{
  private final IntHashSet used;
  private final SecureRandom random;
  private final int min;
  private final int max_count;

  private EchoServerSessionAllocator(
    final int in_min,
    final int in_max,
    final SecureRandom in_random)
  {
    if (in_max < in_min) {
      throw new IllegalArgumentException(
        String.format(
          "Maximum value %d must be >= minimum value %d",
          Integer.valueOf(in_max),
          Integer.valueOf(in_min)));
    }

    this.used = new IntHashSet();
    this.min = in_min;
    this.max_count = Math.max(in_max - in_min, 1);
    this.random = Objects.requireNonNull(in_random, "random");
  }

  /**
   * Create a new session allocator.
   *
   * @param in_min    The minimum session ID (inclusive)
   * @param in_max    The maximum session ID (exclusive)
   * @param in_random A random number generator
   *
   * @return A new allocator
   */

  public static EchoServerSessionAllocator create(
    final int in_min,
    final int in_max,
    final SecureRandom in_random)
  {
    return new EchoServerSessionAllocator(in_min, in_max, in_random);
  }

  /**
   * Allocate a new session.
   *
   * @return A new session ID
   *
   * @throws EchoServerSessionAllocationException If there are no non-allocated sessions left
   */

  public int allocate()
    throws EchoServerSessionAllocationException
  {
    if (this.used.size() == this.max_count) {
      throw new EchoServerSessionAllocationException(
        "No session IDs left to allocate");
    }

    for (int index = 0; index < this.max_count; ++index) {
      final int session = this.random.nextInt(this.max_count) + this.min;
      if (!this.used.contains(session)) {
        this.used.add(session);
        return session;
      }
    }

    throw new EchoServerSessionAllocationException(
      String.format(
        "Unable to allocate a session ID after %d attempts (%d values in use)",
        Integer.valueOf(this.max_count),
        Integer.valueOf(this.used.size()),
        Integer.valueOf(this.max_count)));
  }

  /**
   * Free a session. After this method returns, {@code session} becomes eligible
   * for allocation by future calls to {@link #allocate()}.
   *
   * @param session The session to free
   */

  public void free(final int session)
  {
    this.used.remove(session);
  }
}
```

Класс `ClientState` отвечает за опрос всех текущих _duologues_ для того , чтобы они могли обрабатывать очереди сообщений, а также отвечает за удаление с истекшим сроком годности _duologues_ :

[EchoServer.ClientState.poll\(\)](https://github.com/io7m/aeron-guide/blob/f967e09d36ec8c7f525a66ade2f6f00ab43dcd51/src/main/java/com/io7m/aeron_guide/take2/EchoServer.java#L548)

```java
    public void poll()
    {
      this.exec.assertIsExecutorThread();

      final Iterator<Map.Entry<Integer, EchoServerDuologue>> iter =
        this.client_duologues.entrySet().iterator();

      /*
       * Get the current time; used to expire duologues.
       */

      final Instant now = this.clock.instant();

      while (iter.hasNext()) {
        final Map.Entry<Integer, EchoServerDuologue> entry = iter.next();
        final EchoServerDuologue duologue = entry.getValue();

        final String session_name =
          Integer.toString(entry.getKey().intValue());

        /*
         * If the duologue has either been closed, or has expired, it needs
         * to be deleted.
         */

        boolean delete = false;
        if (duologue.isExpired(now)) {
          LOG.debug("[{}] duologue expired", session_name);
          delete = true;
        }

        if (duologue.isClosed()) {
          LOG.debug("[{}] duologue closed", session_name);
          delete = true;
        }

        if (delete) {
          try {
            duologue.close();
          } finally {
            LOG.debug("[{}] deleted duologue", session_name);
            iter.remove();
            this.port_allocator.free(duologue.portData());
            this.port_allocator.free(duologue.portControl());
            this.address_counter.decrement(duologue.ownerAddress());
          }
          continue;
        }

        /*
         * Otherwise, poll the duologue for activity.
         */

        duologue.poll();
      }
    }
```

Наконец, фактическое состояние и поведение _perduologue_ инкапсулируется классом `EchoServerDuologue`:

[EchoServerDuologue](https://github.com/io7m/aeron-guide/blob/f967e09d36ec8c7f525a66ade2f6f00ab43dcd51/src/main/java/com/io7m/aeron_guide/take2/EchoServerDuologue.java#L29)

```java
public final class EchoServerDuologue implements AutoCloseable
{
  private static final Logger LOG =
    LoggerFactory.getLogger(EchoServerDuologue.class);

  private static final Pattern PATTERN_ECHO =
    Pattern.compile("^ECHO (.*)$");

  private final UnsafeBuffer send_buffer;
  private final EchoServerExecutorService exec;
  private final Instant initial_expire;
  private final InetAddress owner;
  private final int port_data;
  private final int port_control;
  private final int session;
  private final FragmentAssembler handler;
  private boolean closed;
  private Publication publication;
  private Subscription subscription;

  private EchoServerDuologue(
    final EchoServerExecutorService in_exec,
    final Instant in_initial_expire,
    final InetAddress in_owner_address,
    final int in_session,
    final int in_port_data,
    final int in_port_control)
  {
    this.exec =
      Objects.requireNonNull(in_exec, "executor");
    this.initial_expire =
      Objects.requireNonNull(in_initial_expire, "initial_expire");
    this.owner =
      Objects.requireNonNull(in_owner_address, "owner");

    this.send_buffer =
      new UnsafeBuffer(BufferUtil.allocateDirectAligned(1024, 16));

    this.session = in_session;
    this.port_data = in_port_data;
    this.port_control = in_port_control;
    this.closed = false;

    this.handler = new FragmentAssembler((data, offset, length, header) -> {
      try {
        this.onMessageReceived(data, offset, length, header);
      } catch (final IOException e) {
        LOG.error("failed to send message: ", e);
        this.close();
      }
    });
  }

  /**
   * Create a new duologue. This will create a new publication and subscription
   * pair using a specific session ID and intended only for a single client
   * at a given address.
   *
   * @param aeron         The Aeron instance
   * @param clock         A clock used for time-related operations
   * @param exec          An executor
   * @param local_address The local address of the server ports
   * @param owner_address The address of the client
   * @param session       The session ID
   * @param port_data     The data port
   * @param port_control  The control port
   *
   * @return A new duologue
   */

  public static EchoServerDuologue create(
    final Aeron aeron,
    final Clock clock,
    final EchoServerExecutorService exec,
    final InetAddress local_address,
    final InetAddress owner_address,
    final int session,
    final int port_data,
    final int port_control)
  {
    Objects.requireNonNull(aeron, "aeron");
    Objects.requireNonNull(clock, "clock");
    Objects.requireNonNull(exec, "exec");
    Objects.requireNonNull(local_address, "local_address");
    Objects.requireNonNull(owner_address, "owner_address");

    LOG.debug(
      "creating new duologue at {} ({},{}) session {} for {}",
      local_address,
      Integer.valueOf(port_data),
      Integer.valueOf(port_control),
      Integer.toString(session),
      owner_address);

    final Instant initial_expire =
      clock.instant().plus(10L, ChronoUnit.SECONDS);

    final ConcurrentPublication pub =
      EchoChannels.createPublicationDynamicMDCWithSession(
        aeron,
        local_address,
        port_control,
        EchoServer.ECHO_STREAM_ID,
        session);

    try {
      final EchoServerDuologue duologue =
        new EchoServerDuologue(
          exec,
          initial_expire,
          owner_address,
          session,
          port_data,
          port_control);

      final Subscription sub =
        EchoChannels.createSubscriptionWithHandlersAndSession(
          aeron,
          local_address,
          port_data,
          EchoServer.ECHO_STREAM_ID,
          duologue::onClientConnected,
          duologue::onClientDisconnected,
          session);

      duologue.setPublicationSubscription(pub, sub);
      return duologue;
    } catch (final Exception e) {
      try {
        pub.close();
      } catch (final Exception pe) {
        e.addSuppressed(pe);
      }
      throw e;
    }
  }

  /**
   * Poll the duologue for activity.
   */

  public void poll()
  {
    this.exec.assertIsExecutorThread();
    this.subscription.poll(this.handler, 10);
  }

  private void onMessageReceived(
    final DirectBuffer buffer,
    final int offset,
    final int length,
    final Header header)
    throws IOException
  {
    this.exec.assertIsExecutorThread();

    final String session_name =
      Integer.toString(header.sessionId());
    final String message =
      EchoMessages.parseMessageUTF8(buffer, offset, length);

    /*
     * Try to parse an ECHO message.
     */

    LOG.debug("[{}] received: {}", session_name, message);
    final Matcher echo_matcher = PATTERN_ECHO.matcher(message);
    if (echo_matcher.matches()) {
      EchoMessages.sendMessage(
        this.publication,
        this.send_buffer,
        "ECHO " + echo_matcher.group(1));
      return;
    }

    /*
     * Otherwise, fail and close this duologue.
     */

    try {
      EchoMessages.sendMessage(
        this.publication,
        this.send_buffer,
        "ERROR bad message");
    } finally {
      this.close();
    }
  }

  private void setPublicationSubscription(
    final Publication in_publication,
    final Subscription in_subscription)
  {
    this.publication =
      Objects.requireNonNull(in_publication, "Publication");
    this.subscription =
      Objects.requireNonNull(in_subscription, "Subscription");
  }

  private void onClientDisconnected(
    final Image image)
  {
    this.exec.execute(() -> {
      final int image_session = image.sessionId();
      final String session_name = Integer.toString(image_session);
      final InetAddress address = EchoAddresses.extractAddress(image.sourceIdentity());

      if (this.subscription.imageCount() == 0) {
        LOG.debug("[{}] last client ({}) disconnected", session_name, address);
        this.close();
      } else {
        LOG.debug("[{}] client {} disconnected", session_name, address);
      }
    });
  }

  private void onClientConnected(
    final Image image)
  {
    this.exec.execute(() -> {
      final InetAddress remote_address =
        EchoAddresses.extractAddress(image.sourceIdentity());

      if (Objects.equals(remote_address, this.owner)) {
        LOG.debug("[{}] client with correct IP connected",
                  Integer.toString(image.sessionId()));
      } else {
        LOG.error("connecting client has wrong address: {}",
                  remote_address);
      }
    });
  }

  /**
   * @param now The current time
   *
   * @return {@code true} if this duologue has no subscribers and the current
   * time {@code now} is after the intended expiry date of the duologue
   */

  public boolean isExpired(
    final Instant now)
  {
    Objects.requireNonNull(now, "now");

    this.exec.assertIsExecutorThread();

    return this.subscription.imageCount() == 0
      && now.isAfter(this.initial_expire);
  }

  /**
   * @return {@code true} iff {@link #close()} has been called
   */

  public boolean isClosed()
  {
    this.exec.assertIsExecutorThread();

    return this.closed;
  }

  @Override
  public void close()
  {
    this.exec.assertIsExecutorThread();

    if (!this.closed) {
      try {
        try {
          this.publication.close();
        } finally {
          this.subscription.close();
        }
      } finally {
        this.closed = true;
      }
    }
  }

  /**
   * @return The data port
   */

  public int portData()
  {
    return this.port_data;
  }

  /**
   * @return The control port
   */

  public int portControl()
  {
    return this.port_control;
  }

  /**
   * @return The IP address that is permitted to participate in this duologue
   */

  public InetAddress ownerAddress()
  {
    return this.owner;
  }

  /**
   * @return The session ID of the duologue
   */

  public int session()
  {
    return this.session;
  }
}
```

Этот `EchoServerDuologue.onMessageReceived()`метод вызывает ошибки, когда клиент отправляет сообщение, которое сервер не понимает, и завершает работу, закрывая _публикацию_ и _подписку_ . Это устраняет [отмеченные проблемы с сообщениями о плохих клиентах](../client-and-server-part-1/implementation-weaknesses.md#otklyuchenie-servera-echoclient) , эффективно отключая клиентов, которые нарушают протокол.

Использование _динамического MDC_ на _всех клиентов_ каналов _подписки_ и _публикации_ и на _дилогия подписки_ и _публикации_ гарантирует , что клиенты могут подключаться к серверу и получать ответы , даже если эти клиенты находятся за NAT. Это решает [проблемы, отмеченные о NAT](../client-and-server-part-1/implementation-weaknesses.md#ekhoklient-ne-mozhet-byt-pozadi-nat).

Чтобы уменьшить количество дублирующегося кода, возникающего в результате создания _публикаций_ и _подписок_ , мы абстрагируем вызовы, чтобы создать их в своем собственном классе `EchoChannels`:

[EchoChannels](https://github.com/io7m/aeron-guide/blob/f967e09d36ec8c7f525a66ade2f6f00ab43dcd51/src/main/java/com/io7m/aeron_guide/take2/EchoChannels.java#L19)

```java
public final class EchoChannels
{
  private EchoChannels()
  {

  }

  /**
   * Create a publication at the given address and port, using the given
   * stream ID.
   *
   * @param aeron     The Aeron instance
   * @param address   The address
   * @param port      The port
   * @param stream_id The stream ID
   *
   * @return A new publication
   */

  public static ConcurrentPublication createPublication(
    final Aeron aeron,
    final InetAddress address,
    final int port,
    final int stream_id)
  {
    Objects.requireNonNull(aeron, "aeron");
    Objects.requireNonNull(address, "address");

    final String addr_string =
      address.toString().replaceFirst("^/", "");

    final String pub_uri =
      new ChannelUriStringBuilder()
        .reliable(TRUE)
        .media("udp")
        .endpoint(
          new StringBuilder(64)
            .append(addr_string)
            .append(":")
            .append(Integer.toUnsignedString(port))
            .toString())
        .build();

    return aeron.addPublication(pub_uri, stream_id);
  }

  /**
   * Create a subscription with a control port (for dynamic MDC) at the given
   * address and port, using the given stream ID.
   *
   * @param aeron     The Aeron instance
   * @param address   The address
   * @param port      The port
   * @param stream_id The stream ID
   *
   * @return A new publication
   */

  public static Subscription createSubscriptionDynamicMDC(
    final Aeron aeron,
    final InetAddress address,
    final int port,
    final int stream_id)
  {
    Objects.requireNonNull(aeron, "aeron");
    Objects.requireNonNull(address, "address");

    final String addr_string =
      address.toString().replaceFirst("^/", "");

    final String sub_uri =
      new ChannelUriStringBuilder()
        .reliable(TRUE)
        .media("udp")
        .controlEndpoint(
          new StringBuilder(64)
            .append(addr_string)
            .append(":")
            .append(Integer.toUnsignedString(port))
            .toString())
        .controlMode("dynamic")
        .build();

    return aeron.addSubscription(sub_uri, stream_id);
  }

  /**
   * Create a publication with a control port (for dynamic MDC) at the given
   * address and port, using the given stream ID.
   *
   * @param aeron     The Aeron instance
   * @param address   The address
   * @param port      The port
   * @param stream_id The stream ID
   *
   * @return A new publication
   */

  public static ConcurrentPublication createPublicationDynamicMDC(
    final Aeron aeron,
    final InetAddress address,
    final int port,
    final int stream_id)
  {
    Objects.requireNonNull(aeron, "aeron");
    Objects.requireNonNull(address, "address");

    final String addr_string =
      address.toString().replaceFirst("^/", "");

    final String pub_uri =
      new ChannelUriStringBuilder()
        .reliable(TRUE)
        .media("udp")
        .controlEndpoint(
          new StringBuilder(32)
            .append(addr_string)
            .append(":")
            .append(Integer.toUnsignedString(port))
            .toString())
        .controlMode("dynamic")
        .build();

    return aeron.addPublication(pub_uri, stream_id);
  }

  /**
   * Create a subscription at the given address and port, using the given
   * stream ID and image handlers.
   *
   * @param aeron                The Aeron instance
   * @param address              The address
   * @param port                 The port
   * @param stream_id            The stream ID
   * @param on_image_available   Called when an image becomes available
   * @param on_image_unavailable Called when an image becomes unavailable
   *
   * @return A new publication
   */

  public static Subscription createSubscriptionWithHandlers(
    final Aeron aeron,
    final InetAddress address,
    final int port,
    final int stream_id,
    final AvailableImageHandler on_image_available,
    final UnavailableImageHandler on_image_unavailable)
  {
    Objects.requireNonNull(aeron, "aeron");
    Objects.requireNonNull(address, "address");
    Objects.requireNonNull(on_image_available, "on_image_available");
    Objects.requireNonNull(on_image_unavailable, "on_image_unavailable");

    final String addr_string =
      address.toString().replaceFirst("^/", "");

    final String sub_uri =
      new ChannelUriStringBuilder()
        .reliable(TRUE)
        .media("udp")
        .endpoint(
          new StringBuilder(32)
            .append(addr_string)
            .append(":")
            .append(Integer.toUnsignedString(port))
            .toString())
        .build();

    return aeron.addSubscription(
      sub_uri,
      stream_id,
      on_image_available,
      on_image_unavailable);
  }

  /**
   * Create a publication with a control port (for dynamic MDC) at the given
   * address and port, using the given stream ID and session ID.
   *
   * @param aeron     The Aeron instance
   * @param address   The address
   * @param port      The port
   * @param stream_id The stream ID
   * @param session   The session ID
   *
   * @return A new publication
   */

  public static ConcurrentPublication createPublicationDynamicMDCWithSession(
    final Aeron aeron,
    final InetAddress address,
    final int port,
    final int stream_id,
    final int session)
  {
    Objects.requireNonNull(aeron, "aeron");
    Objects.requireNonNull(address, "address");

    final String addr_string =
      address.toString().replaceFirst("^/", "");

    final String pub_uri =
      new ChannelUriStringBuilder()
        .reliable(TRUE)
        .media("udp")
        .controlEndpoint(
          new StringBuilder(32)
            .append(addr_string)
            .append(":")
            .append(Integer.toUnsignedString(port))
            .toString())
        .controlMode("dynamic")
        .sessionId(Integer.valueOf(session))
        .build();

    return aeron.addPublication(pub_uri, stream_id);
  }

  /**
   * Create a subscription at the given address and port, using the given
   * stream ID, session ID, and image handlers.
   *
   * @param aeron                The Aeron instance
   * @param address              The address
   * @param port                 The port
   * @param stream_id            The stream ID
   * @param on_image_available   Called when an image becomes available
   * @param on_image_unavailable Called when an image becomes unavailable
   * @param session              The session ID
   *
   * @return A new publication
   */

  public static Subscription createSubscriptionWithHandlersAndSession(
    final Aeron aeron,
    final InetAddress address,
    final int port,
    final int stream_id,
    final AvailableImageHandler on_image_available,
    final UnavailableImageHandler on_image_unavailable,
    final int session)
  {
    Objects.requireNonNull(aeron, "aeron");
    Objects.requireNonNull(address, "address");
    Objects.requireNonNull(on_image_available, "on_image_available");
    Objects.requireNonNull(on_image_unavailable, "on_image_unavailable");

    final String addr_string =
      address.toString().replaceFirst("^/", "");

    final String sub_uri =
      new ChannelUriStringBuilder()
        .reliable(TRUE)
        .media("udp")
        .endpoint(
          new StringBuilder(32)
            .append(addr_string)
            .append(":")
            .append(Integer.toUnsignedString(port))
            .toString())
        .sessionId(Integer.valueOf(session))
        .build();

    return aeron.addSubscription(
      sub_uri,
      stream_id,
      on_image_available,
      on_image_unavailable);
  }

  /**
   * Create a subscription with a control port (for dynamic MDC) at the given
   * address and port, using the given stream ID, and session ID.
   *
   * @param aeron     The Aeron instance
   * @param address   The address
   * @param port      The port
   * @param stream_id The stream ID
   * @param session   The session ID
   *
   * @return A new publication
   */

  public static Subscription createSubscriptionDynamicMDCWithSession(
    final Aeron aeron,
    final InetAddress address,
    final int port,
    final int session,
    final int stream_id)
  {
    Objects.requireNonNull(aeron, "aeron");
    Objects.requireNonNull(address, "address");

    final String addr_string =
      address.toString().replaceFirst("^/", "");

    final String sub_uri =
      new ChannelUriStringBuilder()
        .reliable(TRUE)
        .media("udp")
        .controlEndpoint(
          new StringBuilder(64)
            .append(addr_string)
            .append(":")
            .append(Integer.toUnsignedString(port))
            .toString())
        .controlMode("dynamic")
        .sessionId(Integer.valueOf(session))
        .build();

    return aeron.addSubscription(sub_uri, stream_id);
  }

  /**
   * Create a publication at the given address and port, using the given
   * stream ID and session ID.
   *
   * @param aeron     The Aeron instance
   * @param address   The address
   * @param port      The port
   * @param stream_id The stream ID
   * @param session   The session ID
   *
   * @return A new publication
   */

  public static ConcurrentPublication createPublicationWithSession(
    final Aeron aeron,
    final InetAddress address,
    final int port,
    final int session,
    final int stream_id)
  {
    Objects.requireNonNull(aeron, "aeron");
    Objects.requireNonNull(address, "address");

    final String addr_string =
      address.toString().replaceFirst("^/", "");

    final String pub_uri =
      new ChannelUriStringBuilder()
        .reliable(TRUE)
        .media("udp")
        .endpoint(
          new StringBuilder(64)
            .append(addr_string)
            .append(":")
            .append(Integer.toUnsignedString(port))
            .toString())
        .sessionId(Integer.valueOf(session))
        .build();

    return aeron.addPublication(pub_uri, stream_id);
  }
}
```

Чтобы решить [проблемы, отмеченные при отправке сообщения](../client-and-server-part-1/implementation-weaknesses.md#otpravka-soobshenii-ne-yavlyaetsya-nadezhnoi) , мы определяем `sendMessage()`метод, который повторяет отправку до ~ 500 мс, прежде чем вызвать исключение. Это гарантирует, что сообщения не могут просто «пропасть»:

[EchoMessages](https://github.com/io7m/aeron-guide/blob/f967e09d36ec8c7f525a66ade2f6f00ab43dcd51/src/main/java/com/io7m/aeron_guide/take2/EchoMessages.java#L18)

```java
public final class EchoMessages
{
  private static final Logger LOG = LoggerFactory.getLogger(EchoMessages.class);

  private EchoMessages()
  {

  }

  /**
   * Send the given message to the given publication. If the publication fails
   * to accept the message, the method will retry {@code 5} times, waiting
   * {@code 100} milliseconds each time, before throwing an exception.
   *
   * @param pub    The publication
   * @param buffer A buffer that will hold the message for sending
   * @param text   The message
   *
   * @return The new publication stream position
   *
   * @throws IOException If the message cannot be sent
   */

  public static long sendMessage(
    final Publication pub,
    final UnsafeBuffer buffer,
    final String text)
    throws IOException
  {
    Objects.requireNonNull(pub, "publication");
    Objects.requireNonNull(buffer, "buffer");
    Objects.requireNonNull(text, "text");

    LOG.trace("[{}] send: {}", Integer.toString(pub.sessionId()), text);

    final byte[] value = text.getBytes(UTF_8);
    buffer.putBytes(0, value);

    long result = 0L;
    for (int index = 0; index < 5; ++index) {
      result = pub.offer(buffer, 0, value.length);
      if (result < 0L) {
        try {
          Thread.sleep(100L);
        } catch (final InterruptedException e) {
          Thread.currentThread().interrupt();
        }
        continue;
      }
      return result;
    }

    throw new IOException(
      "Could not send message: Error code: " + errorCodeName(result));
  }

  private static String errorCodeName(final long result)
  {
    if (result == Publication.NOT_CONNECTED) {
      return "Not connected";
    }
    if (result == Publication.ADMIN_ACTION) {
      return "Administrative action";
    }
    if (result == Publication.BACK_PRESSURED) {
      return "Back pressured";
    }
    if (result == Publication.CLOSED) {
      return "Publication is closed";
    }
    if (result == Publication.MAX_POSITION_EXCEEDED) {
      return "Maximum term position exceeded";
    }
    throw new IllegalStateException();
  }

  /**
   * Extract a UTF-8 encoded string from the given buffer.
   *
   * @param buffer The buffer
   * @param offset The offset from the start of the buffer
   * @param length The number of bytes to extract
   *
   * @return A string
   */

  public static String parseMessageUTF8(
    final DirectBuffer buffer,
    final int offset,
    final int length)
  {
    Objects.requireNonNull(buffer, "buffer");
    final byte[] data = new byte[length];
    buffer.getBytes(offset, data);
    return new String(data, UTF_8);
  }
}
```

Метод сервера `run()`не должен удивлять:  
[EchoServer.run\(\)](https://github.com/io7m/aeron-guide/blob/f967e09d36ec8c7f525a66ade2f6f00ab43dcd51/src/main/java/com/io7m/aeron_guide/take2/EchoServer.java#L229)

```java
  public void run()
  {
    try (final Publication publication = this.setupAllClientsPublication()) {
      try (final Subscription subscription = this.setupAllClientsSubscription()) {

        final FragmentHandler handler =
          new FragmentAssembler(
            (buffer, offset, length, header) ->
              this.onInitialClientMessage(
                publication,
                buffer,
                offset,
                length,
                header));

        while (true) {
          this.executor.execute(() -> {
            subscription.poll(handler, 100);
            this.clients.poll();
          });

          try {
            Thread.sleep(100L);
          } catch (final InterruptedException e) {
            Thread.currentThread().interrupt();
          }
        }
      }
    }
  }
```

[EchoServer.setupAllClientsPublication\(\)](https://github.com/io7m/aeron-guide/blob/f967e09d36ec8c7f525a66ade2f6f00ab43dcd51/src/main/java/com/io7m/aeron_guide/take2/EchoServer.java#L292)

```java
  private Publication setupAllClientsPublication()
  {
    return EchoChannels.createPublicationDynamicMDC(
      this.aeron,
      this.configuration.localAddress(),
      this.configuration.localInitialControlPort(),
      ECHO_STREAM_ID);
  }
```

[EchoServer.setupAllClientsSubscription\(\)](https://github.com/io7m/aeron-guide/blob/f967e09d36ec8c7f525a66ade2f6f00ab43dcd51/src/main/java/com/io7m/aeron_guide/take2/EchoServer.java#L305)

```java
  private Subscription setupAllClientsSubscription()
  {
    return EchoChannels.createSubscriptionWithHandlers(
      this.aeron,
      this.configuration.localAddress(),
      this.configuration.localInitialPort(),
      ECHO_STREAM_ID,
      this::onInitialClientConnected,
      this::onInitialClientDisconnected);
  }
```

