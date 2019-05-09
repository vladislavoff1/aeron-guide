# Концепции

### Publications And Subscriptions

Библиотека Aeron работает с однонаправленными потоками сообщений, известными как _publications_ и _subscriptions_. Интуитивно понятно, что _publication_ - это поток сообщений, в который вы можете _писать_ , а _subscription_ - это поток сообщений, из которого вы можете _читать_ . Это требует небольшой корректировки того, как обычно можно думать о программировании в контексте, скажем, сокетов UDP. UDP-сокеты являются двунаправленными; программист обычно создает сокет UDP, связывает его с локальным адресом, а затем читает из этого сокета, чтобы получать дейтаграммы, и записывает дейтаграммы в тот же сокет для отправки сообщений одноранговым узлам. Aeron, напротив, требует, чтобы программист создал отдельные пары публикаций и подписок для записи и чтения сообщений от одноранговых узлов.

### Media Driver

Aeron определяет протокол, поверх которого пользователь должен реализовать свой собственный протокол сообщений для конкретного приложения. Протокол Aeron обрабатывает детали передачи, повторной передачи, упорядочения, фрагментации и т. д., Оставляя приложение свободным для отправки любых необходимых сообщений, не беспокоясь о том, что эти сообщения действительно приходят и поступают в правильном порядке.

Фактическая низкоуровневая обработка среды передачи \(такой как UDP-сокеты\) обрабатывается модулем кода, известным как _media driver_. Медиа-драйвер может быть запущен как отдельный процесс или может быть встроен в приложение. Для примеров, представленных в этом руководстве, мы предполагаем, что медиа-драйвер будет встроен в приложение, и поэтому приложение отвечает за правильную настройку, запуск и выключение медиа-драйвера.
