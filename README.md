# Введение

[Aeron](https://github.com/real-logic/aeron/) — сверхэффективная библиотека транспорта сообщений для Java и C ++. Он предназначен для работы с ненадежными медиа-протоколами, такими как UDP и Infiniband, и предлагает упорядоченный обмен сообщениями и дополнительную надежность \(путем повторной передачи сообщений в случае пропущенных пакетов\). При разработке и реализации особое внимание уделяется коммуникациям с малой задержкой, что делает библиотеку идеальной для использования в приложениях с требованиями реального времени, такими как быстро развивающиеся сетевые многопользовательские игры, высокочастотная финансовая торговля, VOIP, потоковое видео и т. д. В частности, реализация Java разработана таким образом, что она не будет генерировать мусор во время выполнения в устойчивом состоянии, уменьшая нагрузку на память и работу для сборщика.

Это руководство является попыткой описать, как собрать работающий сервер, который может одновременно обслуживать несколько клиентов. Это несколько смещает перспективу разработчика, использующего Aeron в качестве сетевого компонента клиентского / многопользовательского игрового движка. В частности, предполагается, что описанная конфигурация сервера будет одновременно обслуживать относительно небольшое количество клиентов \(обычно менее ста\), а не одновременно обслуживать десятки тысяч клиентов, чего можно ожидать от высокопроизводительного веб-сервера.

{% hint style="info" %}
Aeron предназначен для обслуживания сотен клиентов, а не тысяч. См. [System Design](https://github.com/real-logic/aeron/wiki/Best-Practices-Guide#system-design) для информации.
{% endhint %}

Это руководство предназначено для дополнения существующего [Java programming guide](https://github.com/real-logic/aeron/wiki/Java-Programming-Guide). Предполагается, что вы знакомы с сетями и сокетами UDP, но опыт работы с Aeron не предполагается. Подробности смотрите в следующих документах:

* [Java programming guide](https://github.com/real-logic/aeron/wiki/Java-Programming-Guide)
* [Aeron protocol specification](https://github.com/real-logic/aeron/wiki/Protocol-Specification)
* [Aeron design overview](https://github.com/real-logic/aeron/wiki/Design-Overview)

Оригинальная статья: [Aeron For The Working Programmer](https://io7m.com/documents/aeron-guide)  


