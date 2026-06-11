# Java Senior Interview Q&A

> Единый большой файл для подготовки: вопросы и развернутые ответы с примерами. Порядок тем — от самых частых на senior Java/Spring собеседованиях к более редким.

## Как отвечать на собеседовании

1. Сначала короткий ответ в 1-2 фразы.
2. Потом механизм: как это работает под капотом.
3. Затем пример из кода, SQL, Kafka или production-кейса.
4. В конце tradeoff: когда подход плох или чем опасен.

## 1. Что делает `@Transactional`?

`@Transactional` объявляет границу транзакции. В Spring это обычно работает через AOP proxy: перед вызовом метода proxy открывает транзакцию, после успешного завершения делает commit, при ошибке — rollback.

Пример:

```java
@Service
@RequiredArgsConstructor
public class PaymentService {
  private final PaymentRepository repository;

  @Transactional
  public void pay(Long paymentId) {
    Payment payment = repository.findById(paymentId).orElseThrow();
    payment.complete();
  }
}
```

Важно: транзакцию открывает не сам метод, а proxy вокруг Spring bean.

## 2. Почему `@Transactional` не работает при self-invocation?

Self-invocation — это вызов метода того же класса через `this`. Такой вызов идет мимо Spring proxy, поэтому AOP advice не срабатывает.

Проблема:

```java
public void outer() {
  inner(); // this.inner(), proxy не участвует
}

@Transactional
public void inner() {
  // транзакция не откроется при вызове из outer()
}
```

Решения: вынести `inner()` в другой bean, использовать `TransactionTemplate` или вызывать метод через self-proxy, если это принято в проекте.

## 3. По каким исключениям Spring откатывает транзакцию?

По умолчанию rollback происходит на `RuntimeException` и `Error`. Checked exceptions не откатывают транзакцию без явного указания.

Пример:

```java
@Transactional(rollbackFor = IOException.class)
public void importFile(Path path) throws IOException {
  // checked IOException теперь приведет к rollback
}
```

Частая ошибка: поймать исключение внутри транзакции, залогировать и не пробросить дальше. Для Spring метод завершился успешно, значит будет commit.

## 4. Какие параметры есть у `@Transactional`?

Основные параметры:

- `propagation` — как метод участвует в существующей транзакции.
- `isolation` — уровень изоляции.
- `readOnly` — подсказка для read-only сценария.
- `timeout` — лимит времени транзакции.
- `rollbackFor` / `noRollbackFor` — правила rollback.
- `transactionManager` — какой менеджер транзакций использовать.

На собесе важно не перечислить параметры, а объяснить последствия: например, `REQUIRES_NEW` приостанавливает текущую транзакцию и открывает новую.

## 5. Чем `REQUIRES_NEW` отличается от `NESTED`?

`REQUIRES_NEW` создает независимую транзакцию. Внешняя транзакция приостанавливается. Commit внутренней транзакции сохранится даже если внешняя потом откатится.

`NESTED` создает savepoint внутри текущей транзакции. Если внешний transaction rollback, откатится и nested-часть.

Пример:

```java
@Transactional
public void outer() {
  auditService.saveAudit(); // REQUIRES_NEW
  throw new RuntimeException();
}
```

Если `saveAudit()` использует `REQUIRES_NEW`, аудит может сохраниться, хотя `outer()` откатится.

## 6. Чем опасен внешний HTTP-вызов внутри транзакции?

Транзакция держит DB connection и, возможно, locks. Если внутри нее вызвать внешний сервис, БД будет ждать сеть. При деградации внешнего сервиса растет latency, блокировки живут дольше, пул соединений исчерпывается.

Лучше:

1. В короткой транзакции сохранить состояние.
2. Опубликовать событие через outbox.
3. Внешний вызов выполнить асинхронно.

## 7. Как защититься от Lost Update?

Lost Update возникает, когда две транзакции читают одно значение и обе записывают результат, перетирая изменение друг друга.

Варианты защиты:

- Оптимистичная блокировка через `@Version`.
- Пессимистичная блокировка через `SELECT FOR UPDATE`.
- Атомарный conditional update.

Пример:

```sql
UPDATE account
SET balance = balance - :amount,
    version = version + 1
WHERE id = :id
  AND version = :version
  AND balance >= :amount;
```

Если обновлено 0 строк, значит был конфликт или недостаточно средств.

## 8. Какие уровни изоляции транзакций нужно знать?

Основные уровни:

- `READ UNCOMMITTED` — возможны dirty reads.
- `READ COMMITTED` — нет dirty read, но возможны non-repeatable read и phantom read.
- `REPEATABLE READ` — повторное чтение одной строки стабильно.
- `SERIALIZABLE` — самый строгий уровень, максимальная цена по конкуренции.

В PostgreSQL дефолт — `READ COMMITTED`. PostgreSQL использует MVCC, поэтому читатели обычно не блокируют писателей.

## 9. Что такое MVCC?

MVCC — multiversion concurrency control. БД хранит версии строк, поэтому транзакция видит снимок данных, подходящий ее уровню изоляции.

Плюс: чтение не блокирует запись. Минус: старые версии строк нужно чистить через vacuum, а долгие транзакции мешают очистке.

## 10. Что такое индекс?

Индекс — структура данных для ускорения поиска. Чаще всего в PostgreSQL используется B-tree. Индекс ускоряет `SELECT`, но замедляет `INSERT`, `UPDATE`, `DELETE`, потому что его тоже нужно обновлять.

Пример:

```sql
CREATE INDEX idx_orders_client_status
ON orders (client_id, status);
```

Такой индекс хорошо работает для условий по `client_id` и по `client_id + status`.

## 11. Что такое правило левой части composite index?

Для индекса `(user_id, status, created_at)` эффективны запросы:

```sql
WHERE user_id = ?
WHERE user_id = ? AND status = ?
WHERE user_id = ? AND status = ? AND created_at > ?
```

Но запрос только по `status` обычно не использует этот индекс эффективно, потому что пропущена первая колонка.

## 12. Почему индекс может не использоваться?

Причины:

- Низкая селективность.
- Устаревшая статистика.
- Маленькая таблица, full scan дешевле.
- Функция над колонкой без functional index.
- Неподходящий порядок колонок в composite index.
- `LIKE '%text'` или `LIKE '%text%'` без специальных индексов.

Пример плохого условия:

```sql
WHERE LOWER(email) = LOWER(:email)
```

Для него нужен functional index:

```sql
CREATE INDEX idx_users_lower_email ON users (LOWER(email));
```

## 13. Что такое селективность индекса?

Селективность — насколько хорошо колонка отфильтровывает строки. Если в таблице 2 млн строк и почти все `id` уникальны, индекс по `id` полезен. Если колонка `gender` имеет 2 значения, индекс часто бесполезен.

Высокая селективность — хороша для индекса. Низкая — оптимизатор может выбрать full scan.

## 14. Как искать медленные запросы в PostgreSQL?

Практический порядок:

1. `pg_stat_statements` — найти топ по total time, mean time, calls.
2. `EXPLAIN ANALYZE` — увидеть реальный план.
3. `pg_stat_activity` — проверить активные запросы и блокировки.
4. Проверить индексы, cardinality, outdated statistics.

Пример:

```sql
EXPLAIN ANALYZE
SELECT *
FROM orders
WHERE client_id = 10
ORDER BY created_at DESC
LIMIT 20;
```

## 15. Что такое `SELECT FOR UPDATE`?

`SELECT FOR UPDATE` блокирует выбранные строки до конца транзакции. Другие транзакции не смогут изменить эти строки, пока lock не освобожден.

Пример:

```sql
BEGIN;
SELECT * FROM account WHERE id = 1 FOR UPDATE;
UPDATE account SET balance = balance - 100 WHERE id = 1;
COMMIT;
```

Это подходит для денежных операций, но может снижать throughput при высокой конкуренции.

## 16. Для чего нужен `SKIP LOCKED`?

`SKIP LOCKED` пропускает уже заблокированные строки. Это удобно для очереди задач в БД: несколько worker'ов берут разные задачи и не ждут друг друга.

Пример:

```sql
SELECT id
FROM job
WHERE status = 'NEW'
ORDER BY created_at
FOR UPDATE SKIP LOCKED
LIMIT 100;
```

## 17. Что такое ACID?

ACID:

- Atomicity — все или ничего.
- Consistency — данные остаются валидными.
- Isolation — параллельные транзакции не ломают друг друга.
- Durability — commit не теряется после сбоя.

Durability обычно обеспечивается WAL: сначала запись в журнал, потом изменение файлов данных.

## 18. Как реализовать many-to-many в реляционной БД?

Через таблицу-связку с двумя foreign key.

```sql
CREATE TABLE user_role (
  user_id BIGINT NOT NULL REFERENCES users(id),
  role_id BIGINT NOT NULL REFERENCES roles(id),
  PRIMARY KEY (user_id, role_id)
);
```

На FK часто нужны индексы, потому что сама FK-связь не всегда автоматически создает нужный индекс.

## 19. Что такое шардирование?

Шардирование — горизонтальное разделение данных по shard key. Например, пользователи с разными `user_id` лежат в разных БД.

Нужны:

- shard key;
- router;
- mapping;
- rebalancing;
- защита от hot shard;
- стратегия cross-shard запросов.

Шардирование сложно откатывать, поэтому до него обычно используют индексы, read replicas, partitioning и caching.

## 20. Что такое JMM?

Java Memory Model описывает, как потоки видят память: visibility, atomicity, ordering. Без синхронизации один поток может не увидеть изменение другого или увидеть операции в другом порядке.

Пример проблемы:

```java
class Flag {
  boolean stopped;

  void stop() {
    stopped = true;
  }

  void run() {
    while (!stopped) {
      // может крутиться бесконечно
    }
  }
}
```

Поле `stopped` должно быть `volatile` или доступ должен быть синхронизирован.

## 21. Что дает `volatile`?

`volatile` дает видимость и happens-before между записью и последующим чтением. Но не дает атомарность.

Плохо:

```java
private volatile int count;

public void increment() {
  count++; // read -> add -> write, операция не атомарна
}
```

Правильно:

```java
private final AtomicInteger count = new AtomicInteger();

public void increment() {
  count.incrementAndGet();
}
```

## 22. Чем `volatile` отличается от `synchronized`?

`volatile` — видимость без блокировки. `synchronized` — взаимное исключение, атомарность и видимость через monitor.

`volatile` подходит для флагов остановки и safe publication. Для инкремента счетчика, изменения нескольких полей или инвариантов нужен lock, atomic или другой механизм синхронизации.

## 23. Что такое happens-before?

Happens-before — гарантия, что результат одной операции виден другой операции.

Примеры:

- запись в `volatile` happens-before чтения этого `volatile`;
- unlock monitor happens-before следующего lock этого monitor;
- `Thread.start()` happens-before код в новом потоке;
- действия в потоке happens-before успешного `Thread.join()`;
- запись final-полей в конструкторе видна после корректной публикации объекта.

## 24. Как работает CAS?

CAS — compare-and-swap. Операция атомарно проверяет, что текущее значение равно ожидаемому, и заменяет его новым.

Пример:

```java
AtomicInteger value = new AtomicInteger(0);
value.compareAndSet(0, 1);
```

Минусы CAS: busy-spin под конкуренцией, ABA-проблема, отсутствие fairness, неудобство для составных операций.

## 25. Почему `containsKey + put` в `ConcurrentHashMap` не атомарно?

Между `containsKey()` и `put()` другой поток может изменить map.

Плохо:

```java
if (!cache.containsKey(key)) {
  cache.put(key, load(key));
}
```

Лучше:

```java
cache.computeIfAbsent(key, this::load);
```

## 26. Почему `ConcurrentHashMap` не допускает `null`?

Чтобы не было неоднозначности: `map.get(key) == null` означало бы либо "ключа нет", либо "ключ есть, значение null". В concurrent-среде такая неоднозначность ломает корректные алгоритмы.

## 27. Что такое `ReadWriteLock`?

`ReadWriteLock` разделяет доступ на read lock и write lock. Несколько читателей могут работать одновременно, писатель — эксклюзивно.

Подходит, когда чтений много, записей мало. При частых записях может быть хуже обычного lock.

## 28. Что такое virtual threads в Java 21?

Virtual threads — легкие потоки JVM, полезные для blocking I/O. Они упрощают thread-per-request модель и уменьшают стоимость большого числа ожидающих операций.

Ограничения:

- не ускоряют CPU-bound код;
- не увеличивают параллелизм выше числа CPU;
- не заменяют backpressure;
- в чистом WebFlux/R2DBC часто не дают главного выигрыша.

## 29. Почему `synchronized` не работает между несколькими pod'ами?

`synchronized` блокирует только потоки внутри одной JVM. Если приложение запущено в 5 pod'ах, у каждого pod свой heap и свои monitor locks.

Для распределенной блокировки используют БД, Redis, ZooKeeper, etcd или специализированные решения.

Пример DB lock:

```sql
UPDATE locks
SET lock_until = now() + interval '30 seconds'
WHERE name = 'daily-job'
  AND lock_until <= now();
```

Если обновлена 1 строка — lock захвачен.

## 30. Как остановить поток корректно?

Варианты:

- `volatile` flag для собственного цикла;
- `Thread.interrupt()`;
- `ExecutorService.shutdown()`;
- cooperative cancellation в `CompletableFuture`/reactive pipeline.

`Thread.stop()`, `suspend()`, `resume()` использовать нельзя: они могут оставить объект в неконсистентном состоянии.

## 31. Что такое deadlock, livelock и starvation?

Deadlock — потоки ждут lock'и друг друга и не могут продолжить. Livelock — потоки активны, но постоянно мешают друг другу. Starvation — поток долго не получает ресурс.

Профилактика deadlock:

- фиксированный порядок взятия lock'ов;
- timeout через `tryLock`;
- меньше вложенных lock'ов;
- короткие критические секции.

## 32. Как устроен `HashMap`?

`HashMap` хранит массив бакетов. Индекс бакета рассчитывается по hash. Коллизии сначала хранятся в linked list, при большом числе элементов в бакете превращаются в tree bin.

Ключевой контракт: если `a.equals(b) == true`, то `a.hashCode() == b.hashCode()`.

## 33. Что будет, если переопределить `equals`, но не `hashCode`?

Объекты могут считаться равными через `equals`, но попадать в разные бакеты в `HashMap`/`HashSet`. В итоге поиск и удаление работают некорректно.

Пример:

```java
record UserId(String value) {
}
```

`record` хорош тем, что корректно генерирует `equals` и `hashCode`.

## 34. Почему mutable key в `HashMap` опасен?

Если поля, участвующие в `hashCode`, изменились после помещения ключа в map, объект останется в старом бакете, а искать его будут по новому hash.

Итог: ключ "теряется".

## 35. Что такое `Integer` cache?

Для значений `-128..127` JVM кеширует boxed `Integer`. Поэтому:

```java
Integer a = 127;
Integer b = 127;
System.out.println(a == b); // true

Integer x = 128;
Integer y = 128;
System.out.println(x == y); // false
```

Обертки нужно сравнивать через `equals`, не через `==`.

## 36. Что такое `ConcurrentModificationException`?

Fail-fast исключение при структурном изменении коллекции во время итерации.

Плохо:

```java
for (User user : users) {
  users.remove(user);
}
```

Правильно:

```java
users.removeIf(User::isBlocked);
```

Или `Iterator.remove()`.

## 37. Как устроена память JVM?

Основные области:

- Heap — объекты.
- Stack — frames вызовов методов каждого потока.
- Metaspace — metadata классов.
- PC Register.
- Native Method Stack.

Локальные примитивы обычно в stack frame, объекты — в heap, ссылки на них могут быть в stack.

## 38. Как работает GC?

На высоком уровне GC ищет живые объекты от GC Roots, помечает достижимые, удаляет недостижимые и при необходимости compact'ит память.

Young generation обычно собирается чаще. Объекты проходят Eden -> Survivor -> Old, если живут достаточно долго.

## 39. Какие бывают ссылки в Java?

- Strong — обычная ссылка, объект не собирается.
- Soft — может быть собрана при нехватке памяти.
- Weak — собирается при ближайшем GC, если нет strong-ссылок.
- Phantom — используется для post-mortem cleanup через reference queue.

## 40. Почему добавление RAM не всегда лечит OOM?

Если есть memory leak или unbounded cache, увеличение heap только откладывает падение. Нужно найти источник удержания ссылок: growing collection, static cache, listener leak, ThreadLocal, queue backlog.

## 41. Как работают ClassLoader'ы?

Основная цепочка:

- Bootstrap.
- Platform.
- Application.

Parent delegation: сначала запрос на загрузку класса делегируется родителю. Класс уникален парой: fully qualified name + classloader.

## 42. Что такое String pool?

String pool хранит интернированные строки. Литералы попадают туда автоматически, `intern()` помещает строку в пул или возвращает существующую.

```java
String a = "java";
String b = "ja" + "va";
System.out.println(a == b); // true, compile-time constant
```

## 43. Как устроен жизненный цикл Spring bean?

Упрощенно:

1. Чтение `BeanDefinition`.
2. Создание объекта.
3. Dependency injection.
4. `Aware` callbacks.
5. `BeanPostProcessor#postProcessBeforeInitialization`.
6. Init: `@PostConstruct`, `afterPropertiesSet`.
7. `BeanPostProcessor#postProcessAfterInitialization`.
8. Использование.
9. Destroy callbacks.

## 44. Constructor injection vs field injection?

Constructor injection предпочтительнее:

- зависимость обязательна и видна в API класса;
- можно сделать поля `final`;
- проще тестировать;
- нет скрытой магии reflection.

Field injection усложняет тесты и делает объект невалидным без Spring.

## 45. JDK Proxy vs CGLIB?

JDK Dynamic Proxy проксирует интерфейсы. CGLIB создает subclass целевого класса. В Spring Boot CGLIB часто включен по умолчанию для классов без интерфейсов.

Ограничения CGLIB: `final` классы и `final` методы нельзя переопределить.

## 46. Что такое `DispatcherServlet`?

`DispatcherServlet` — front controller Spring MVC. Он принимает HTTP-запрос, через `HandlerMapping` находит controller, вызывает handler adapter, обрабатывает результат и возвращает response.

Путь запроса:

```text
Client -> Nginx -> Tomcat/Netty -> Filter chain -> DispatcherServlet -> Controller -> Service -> Repository -> DB
```

## 47. Что такое Spring Boot starter?

Starter — модуль, который приносит зависимости и автоконфигурацию. В Spring Boot 3 автоконфигурации регистрируются через:

```text
META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports
```

Обычно используются `@AutoConfiguration`, `@ConditionalOnClass`, `@ConditionalOnMissingBean`, `@ConfigurationProperties`.

## 48. Как выбирать между JdbcTemplate, JPA, MyBatis и R2DBC?

- `JdbcTemplate` — полный контроль над SQL.
- JPA/Hibernate — быстрая CRUD-разработка и богатая модель связей.
- MyBatis — SQL руками, mapping удобнее чем JDBC.
- R2DBC — reactive non-blocking доступ к БД.

Если нужен сложный SQL и высокая предсказуемость, JPA может быть хуже JDBC/jOOQ/MyBatis.

## 49. Что такое N+1 в Hibernate?

N+1 — один запрос получает список сущностей, затем для каждой сущности выполняется отдельный запрос за связью.

Лечение:

```java
@Query("select u from User u join fetch u.roles where u.active = true")
List<User> findActiveWithRoles();
```

Другие варианты: `@EntityGraph`, batch fetch, DTO projection.

## 50. Чем `LAZY` отличается от `EAGER`?

`LAZY` грузит связь при обращении. `EAGER` грузит сразу. `EAGER` часто опасен: может тянуть лишние данные и провоцировать сложные запросы или N+1.

Для API часто лучше DTO projection с явным fetch-планом.

## 51. Когда возникает `LazyInitializationException`?

Когда lazy-связь читают после закрытия persistence context/session.

Плохо:

```java
User user = userRepository.findById(id).orElseThrow();
return user.getRoles().size(); // если вне транзакции, может упасть
```

Решения: fetch join, entity graph, DTO query, работа внутри транзакции.

## 52. Почему `@Data` на JPA entity опасна?

`@Data` генерирует `equals`, `hashCode`, `toString` по всем полям. Для entity это опасно:

- lazy-связи могут загрузиться случайно;
- возможна рекурсия в `toString`;
- `equals/hashCode` ломаются с proxy и mutable id.

Для entity лучше явно писать `equals/hashCode`, обычно по immutable business key или аккуратно по id.

## 53. Что такое Kafka?

Kafka — распределенный commit log. Producer пишет records в topic, topic разделен на partitions, consumers читают records по offset.

Ключевая гарантия: порядок есть только внутри одной partition.

## 54. Что такое consumer group?

Consumer group — группа consumer'ов, которые совместно читают topic. Каждая partition назначается только одному consumer внутри группы. Разные группы читают независимо.

Если consumer'ов больше, чем partitions, лишние consumer'ы будут idle.

## 55. Что такое Kafka lag?

Lag — разница между последним offset в partition и offset, который обработал consumer.

Рост lag означает, что consumer не успевает. Причины: медленный downstream, мало partitions/consumer'ов, тяжелая обработка, ошибки и retries.

## 56. Какие гарантии доставки есть в Kafka?

- At most once — можно потерять, но без дублей.
- At least once — не теряем, но возможны дубли.
- Exactly once — сложная комбинация идемпотентного producer, transactions и корректной обработки.

В production часто используют at-least-once + идемпотентный consumer.

## 57. Почему ручной commit offset лучше auto-commit?

Auto-commit может сохранить offset до завершения обработки. Если сервис упал после commit, сообщение потеряно.

При ручном commit offset подтверждается только после успешной обработки.

Пример логики:

```java
process(record)
    .doOnSuccess(unused -> record.receiverOffset().acknowledge());
```

## 58. Как делать идемпотентный Kafka consumer?

Варианты:

- `idempotency_key` и unique constraint в БД;
- таблица processed messages;
- Redis `SET NX` с TTL;
- операции, которые по природе дают одинаковый результат при повторе.

Пример:

```sql
CREATE UNIQUE INDEX ux_payment_event_id
ON payment_event (event_id);
```

Повторное сообщение упадет на unique constraint, consumer поймет, что событие уже обработано.

## 59. Что такое DLQ/DLT?

Dead Letter Queue/Topic — место для сообщений, которые не удалось обработать после retry.

В Spring Kafka:

```java
factory.setCommonErrorHandler(new DefaultErrorHandler(
    new DeadLetterPublishingRecoverer(kafkaTemplate),
    new FixedBackOff(1000L, 3)
));
```

Так poison message не блокирует partition бесконечно.

## 60. Что такое Transactional Outbox?

Outbox решает проблему: "как сохранить данные в БД и гарантированно отправить событие в Kafka".

Схема:

1. В одной DB transaction сохранить бизнес-данные и строку в `outbox`.
2. Отдельный worker или Debezium читает outbox.
3. Публикует событие в Kafka.
4. Помечает событие отправленным или полагается на CDC.

Так нет distributed transaction между БД и Kafka.

## 61. Как Kafka гарантирует порядок?

Порядок гарантирован только внутри partition. Если нужен порядок по пользователю, ключом сообщения должен быть `userId`, чтобы события пользователя попадали в одну partition.

Если обработку распараллелить после чтения, порядок можно сломать даже внутри одной partition.

## 62. Что такое `acks` у Kafka producer?

- `acks=0` — producer не ждет подтверждения.
- `acks=1` — ждет leader.
- `acks=all` — ждет все in-sync replicas.

Для надежной отправки обычно используют `acks=all`, retries и `enable.idempotence=true`.

## 63. Как устроен Event Orchestrator из примера?

Pipeline:

```text
Kafka input topic
-> listener
-> lookup processor definitions by event type
-> FILTER
-> MAPPER
-> ROUTER
-> Kafka output topic
```

Идея: логика обработки вынесена в конфигурацию в БД, а не зашита в код. Новый тип события можно добавить конфигом и reload endpoint'ом.

## 64. Какие проблемы были у Kafka pipeline под нагрузкой?

Типовые проблемы:

- JEXL expression compilation на каждый event.
- Race condition при reload конфигурации.
- Логирование полного payload.
- Unbounded virtual threads.
- Рост lag на тяжелых mapper'ах.

Решения: кэшировать compiled expressions, copy-on-write для конфигурации, логировать metadata вместо payload, лимитировать параллелизм.

## 65. Почему WebFlux, а не Spring MVC?

WebFlux полезен для IO-bound систем с высокой конкуренцией: БД, Redis, Kafka, HTTP, object storage. Он держит много concurrent requests на небольшом количестве event loop threads.

Но это работает только если не блокировать event loop. Один blocking MinIO/JDBC вызов может испортить latency соседних запросов.

## 66. Чем R2DBC отличается от JDBC?

JDBC блокирует поток на время запроса к БД. R2DBC возвращает `Mono`/`Flux` и не блокирует поток ожиданием I/O.

Если приложение уже WebFlux, JDBC внутри reactive chain ломает модель.

## 67. Как не блокировать event loop?

Правило: blocking I/O уводить на `boundedElastic`, CPU-bound на `parallel` или отдельный scheduler, event loop оставлять для неблокирующих операций.

Пример:

```java
Mono.fromCallable(() -> minioClient.putObject(args))
    .subscribeOn(Schedulers.boundedElastic());
```

## 68. Что такое backpressure в Reactor?

Backpressure — контроль скорости producer'а относительно consumer'а. В прикладном коде часто проявляется как лимит concurrency.

Пример:

```java
Flux.fromIterable(items)
    .flatMap(this::saveToDb, 100);
```

Одновременно будет не больше 100 операций сохранения.

## 69. `flatMap` vs `concatMap`?

`flatMap` выполняет inner publishers конкурентно и не гарантирует порядок. `concatMap` выполняет последовательно и сохраняет порядок.

Если важен throughput — `flatMap` с лимитом. Если важен порядок — `concatMap`.

## 70. Что такое hot и cold publisher?

Cold publisher начинает работу для каждого subscriber заново. Hot publisher производит данные независимо от конкретного subscriber.

HTTP-запрос через `WebClient` обычно cold: пока нет subscription, запрос не выполняется.

## 71. Почему reactive chain не выполняется без subscribe?

Reactor ленивый. Операторы только описывают pipeline. Выполнение стартует при subscription.

В controller обычно не нужно вручную вызывать `subscribe()`: нужно вернуть `Mono` или `Flux` наружу, и Spring подпишется сам.

## 72. Как работают reactive transactions?

В R2DBC транзакционный контекст хранится не в `ThreadLocal`, а в Reactor Context. Поэтому переключение scheduler'ов внутри транзакции может быть опасным, если ломает контекст.

Лучше использовать `TransactionalOperator` для явных reactive chains.

## 73. Как обрабатывать большие Excel-файлы без OOM?

Нужно streaming processing, а не загрузка всего файла в память.

Подход:

- Apache POI Streaming/SAX reader.
- Batch processing.
- Ограничение размера файла.
- Временные файлы для генерации.
- Offload на `boundedElastic`.

## 74. Что такое cache-aside?

Cache-aside:

1. Проверяем cache.
2. Если miss — читаем source of truth.
3. Кладем в cache с TTL.
4. Возвращаем результат.

Проблема: thundering herd при одновременном cache miss. Решение: lock через Redis `SET NX`, request coalescing или preload.

## 75. Что такое Redis fast path?

Fast path — когда наиболее частый read-сценарий обслуживается из Redis без обращения к БД или внешнему API.

Это снижает latency и нагрузку на source of truth, но требует TTL, invalidation strategy и fallback.

## 76. Что такое circuit breaker?

Circuit breaker защищает сервис от деградации downstream. Если внешний сервис часто падает или тормозит, breaker открывается и быстрым образом возвращает fallback/error, не накапливая тысячи висящих запросов.

Связанные паттерны: timeout, retry, bulkhead, rate limit.

## 77. Retry: чем опасен?

Retry может усилить аварию. Если downstream уже перегружен, агрессивные retries увеличат нагрузку.

Правильно: retry только для transient errors, ограниченное число попыток, backoff + jitter, circuit breaker, idempotency.

## 78. Какие bottleneck'и были в highload-примерах?

Основные:

- blocking MinIO в reactive chain;
- Oracle/R2DBC pool exhaustion;
- внешние API без circuit breaker;
- Redis cache miss thundering herd;
- nginx backlog при большом concurrency;
- тяжелая JEXL/JSON обработка.

Сильный ответ: "система IO-bound, поэтому потолок определялся внешними зависимостями и очередями, а не CPU".

## 79. Какие метрики важны для highload?

Ключевые:

- p95/p99 latency;
- error rate;
- RPS/throughput;
- DB pool acquired/pending;
- Redis latency;
- Kafka lag;
- heap usage и GC pauses;
- circuit breaker state;
- nginx/connect timeouts.

CPU важен, но для IO-bound сервиса часто не главный лимит.

## 80. Как проводить load testing?

Нужно тестировать разные режимы:

- smoke — проверка доступности;
- stress — реалистичная нагрузка;
- spike — резкий всплеск;
- breaking point — поиск потолка;
- soak — долгий тест на memory leak.

Смотреть надо не только RPS, но и p95/p99, errors, saturation ресурсов и recovery после деградации.

## 81. Какие результаты нагрузочного теста можно использовать в ответе?

Из материалов:

- Advertising stress: около `2 697 RPS`, `p95 = 57ms`, ошибок `0%`.
- Advertising peak: около `3 900 RPS`, деградация после `3 000-3 500 concurrent`.
- In-app delivery stress: около `1 370 RPS`, `p95 = 34ms`, ошибок `0%`.
- In-app delivery breaking point: около `800-1000 concurrent`, `~1800 RPS`, дальше connect timeouts.

Важно говорить, что цифры зависят от endpoint, стенда, cache hit ratio и профиля нагрузки.

## 82. Что значит IO-bound система?

IO-bound значит, что время уходит на ожидание I/O: БД, Redis, Kafka, HTTP, MinIO. Увеличение CPU может почти не дать прироста, если узкое место — пул соединений, внешний API или очередь ingress.

CPU-bound задачи: JSON parsing, JEXL, Excel generation, compression. Их нужно профилировать отдельно.

## 83. Как профилировать приложение?

Практический набор:

- Prometheus/Grafana — latency, errors, pools, GC.
- Tracing — найти медленный span.
- Pyroscope/JFR — CPU, allocations, locks.
- Logs с correlation id.
- `EXPLAIN ANALYZE` для SQL.

В reactive локально можно использовать `Hooks.onOperatorDebug()` или точечные `.checkpoint("name")`, но в production это дорого.

## 84. Что такое observability?

Observability — способность понять состояние системы по внешним сигналам:

- logs;
- metrics;
- traces.

Для distributed tracing важны Trace ID и Span ID. OpenTelemetry стандартизирует сбор и передачу контекста.

## 85. Что такое REST?

REST — архитектурный стиль: ресурсы, HTTP methods, stateless взаимодействие, uniform interface.

Пример:

```text
GET /users/10
POST /users
PUT /users/10
PATCH /users/10
DELETE /users/10
```

GET не должен создавать ресурс, потому что GET safe и идемпотентен.

## 86. PUT vs PATCH?

`PUT` обычно заменяет ресурс целиком и идемпотентен. `PATCH` применяет частичное изменение и не всегда идемпотентен.

Пример:

```http
PUT /users/10
{"name":"Ivan","email":"a@b.com"}

PATCH /users/10
{"email":"new@b.com"}
```

## 87. Как делать идемпотентность HTTP API?

Варианты:

- `Idempotency-Key` header;
- unique constraint;
- conditional update;
- ETag / `If-Match`;
- business operation id.

Для платежей idempotency key обязателен: повтор запроса клиентом не должен списать деньги дважды.

## 88. Как централизованно обрабатывать ошибки в Spring?

Через `@RestControllerAdvice` и `@ExceptionHandler`.

```java
@RestControllerAdvice
class GlobalExceptionHandler {
  @ExceptionHandler(NotFoundException.class)
  ResponseEntity<ErrorResponse> handle(NotFoundException ex) {
    return ResponseEntity.status(HttpStatus.NOT_FOUND)
        .body(new ErrorResponse("NOT_FOUND", ex.getMessage()));
  }
}
```

## 89. REST vs gRPC?

REST/JSON проще для публичных API, browser-friendly, легче дебажить. gRPC быстрее, имеет строгий protobuf contract, поддерживает streaming и хорош для internal service-to-service.

Минус gRPC: сложнее для браузера и ручной отладки.

## 90. Какие типы gRPC вызовов есть?

- Unary.
- Server streaming.
- Client streaming.
- Bidirectional streaming.

Streaming полезен для потоков событий, больших выгрузок и realtime-сценариев.

## 91. Stream API: что важно знать?

Stream — ленивый одноразовый pipeline. Intermediate operations (`map`, `filter`) ленивые, terminal operations (`collect`, `toList`, `count`) запускают выполнение.

После terminal operation stream нельзя использовать повторно.

## 92. Почему `parallelStream` опасен?

`parallelStream` использует общий `ForkJoinPool`. Он может конфликтовать с другими задачами, плохо подходит для blocking I/O и может ухудшить производительность на маленьких коллекциях.

Для I/O лучше явный executor или virtual threads. Для CPU-bound — измерять.

## 93. Где использовать `Optional`?

`Optional` хорошо использовать как return type, когда значение может отсутствовать.

Не стоит использовать `Optional` в полях entity, DTO и параметрах методов без сильной причины.

Пример:

```java
Optional<User> findByEmail(String email);
```

## 94. Checked vs unchecked exceptions?

Checked exceptions заставляют вызывающий код обработать или пробросить ошибку. Минусы: засоряют сигнатуры, плохо сочетаются с lambdas, часто провоцируют бессмысленный catch.

В Spring бизнесовые ошибки часто делают unchecked и обрабатывают централизованно.

## 95. Как работает try-with-resources?

try-with-resources автоматически закрывает ресурсы, реализующие `AutoCloseable`. Если исключение возникло и в try-блоке, и при закрытии, основным будет исключение из try, а ошибка закрытия попадет в suppressed.

```java
try (InputStream in = Files.newInputStream(path)) {
  return in.readAllBytes();
}
```

## 96. Что такое SOLID?

Кратко:

- SRP — один класс, одна причина изменения.
- OCP — расширяем без изменения существующего кода.
- LSP — наследник заменяет родителя без нарушения контракта.
- ISP — узкие интерфейсы лучше толстых.
- DIP — зависим от абстракций, не от деталей.

Важно: SOLID не цель сам по себе. Перегиб приводит к overengineering.

## 97. Factory + Strategy: где применять?

Когда есть набор алгоритмов или обработчиков по типу события/команды.

Пример:

```java
public interface EventHandler {
  EventType type();

  void handle(Event event);
}
```

При старте можно собрать `Map<EventType, EventHandler>`. Добавление нового обработчика не требует `switch`.

## 98. Proxy pattern vs Spring AOP proxy?

Proxy pattern дает объект-заместитель, который контролирует доступ к целевому объекту. Spring AOP proxy — практическая реализация этой идеи для cross-cutting concerns: transactions, security, metrics, caching.

Ограничения Spring proxy: self-invocation, final/private методы, создание объекта через `new`.

## 99. Как реализовать безопасный money transfer?

Требования:

- сумма положительная;
- отправитель и получатель существуют;
- нельзя переводить самому себе;
- операция атомарна;
- защита от lost update;
- idempotency key для повторов;
- audit.

Пример подхода:

```sql
UPDATE account
SET balance = balance - :amount
WHERE id = :fromId
  AND balance >= :amount;

UPDATE account
SET balance = balance + :amount
WHERE id = :toId;
```

Обе операции должны быть в одной транзакции. Для высокой конкуренции добавить version или row lock.

## 100. Как ревьюить сервис с внешним клиентом?

Искать:

- timeout;
- retry с backoff;
- circuit breaker;
- bulkhead;
- логирование correlation id;
- не вызывается ли внешний сервис внутри DB transaction;
- корректная обработка 4xx/5xx;
- нет ли `new WebClient()`/`new ObjectMapper()` в hot path.

## 101. Как ревьюить JPA entity?

Искать:

- `@Data` на entity;
- primitive `long id` вместо nullable `Long`;
- нет `@GeneratedValue`;
- опасный `CascadeType.ALL`;
- lazy-связи в `toString`;
- mutable коллекции без инициализации;
- `equals/hashCode` по изменяемым полям;
- нет индексов на частые query patterns.

## 102. Как ревьюить scheduled job на нескольких pod'ах?

Проблемы:

- все pod'ы берут одну и ту же работу;
- нет distributed lock;
- внешний вызов внутри транзакции;
- статус `processed=true` ставится до успешной отправки;
- нет retry/DLQ;
- нет `PROCESSING/FAILED` статусов.

Решение: batch selection через `FOR UPDATE SKIP LOCKED`, короткая транзакция, статусы, retry и idempotency.

## 103. Как ревьюить thread-safe cache?

Искать:

- `HashMap` вместо `ConcurrentHashMap`;
- mutable key вроде `byte[]`;
- возвращается mutable value без defensive copy;
- тяжелая операция внутри `synchronized`;
- нет TTL/max size;
- cache stampede.

Для production лучше Caffeine/Redis, а не самописный unbounded cache.

## 104. Как объяснить `@Value` и constructor injection проблему?

Если класс имеет несколько конструкторов или no-args constructor, Spring может выбрать не тот путь создания. `@Value` на final поле вместе с лишним constructor часто приводит к ошибкам.

Лучше:

```java
@ConfigurationProperties(prefix = "client")
public record ClientProperties(String baseUrl, Duration timeout) {
}
```

## 105. Что такое Bean Validation и где использовать?

Bean Validation валидирует входные DTO декларативно.

```java
public record TransferRequest(
    @NotNull Long fromAccountId,
    @NotNull Long toAccountId,
    @Positive BigDecimal amount
) {
}
```

В controller нужен `@Valid`. Это лучше ручных `if` в каждом методе.

## 106. Как делать zero-downtime deployment?

Подход:

- rolling update;
- readiness/liveness probes;
- graceful shutdown;
- backward-compatible DB migrations;
- feature flags;
- canary или blue-green для рискованных релизов.

Для БД часто используют expand-contract: сначала добавить новое поле, потом перевести код, потом удалить старое.

## 107. Что такое Docker vs Kubernetes?

Docker — упаковка и запуск контейнера. Kubernetes — оркестрация: scheduling, scaling, self-healing, service discovery, rolling updates.

Основные сущности: Pod, Deployment, Service, ConfigMap, Secret, Ingress.

## 108. Что такое Ingress и Egress?

Ingress — входящий HTTP/HTTPS трафик в кластер. Egress — исходящий трафик из pod'ов наружу.

На highload ingress может стать bottleneck из-за backlog, TLS, timeouts и лимитов соединений.

## 109. Что такое OAuth 2.0 grant types?

Актуальные:

- Authorization Code + PKCE — user login.
- Client Credentials — service-to-service.
- Refresh Token — обновление access token.

Password и Implicit считаются устаревшими для большинства новых систем.

## 110. Что такое цифровая подпись?

Подпись доказывает автора и неизменность данных. Обычно считается hash сообщения, затем hash подписывается приватным ключом. Проверка выполняется публичным ключом.

Это не то же самое, что шифрование для секретности.

## 111. Как проектировать систему балансов?

Ключевые решения:

- один authoritative source of truth;
- idempotency key на операции;
- транзакционная запись ledger;
- optimistic/pessimistic locking;
- audit trail;
- reconciliation;
- метрики расхождения балансов;
- защита от replay.

Лучше хранить не только текущий баланс, но и журнал операций.

## 112. Как отвечать про highload систему на 1500 RPS?

Не говорить просто "система выдерживает 1500 RPS". Нужно уточнить:

- какие endpoints;
- read/write ratio;
- cache hit ratio;
- p95/p99;
- error rate;
- concurrency;
- стенд и ресурсы;
- bottleneck.

Сильная формулировка: "1500 RPS был пик на read-path, большинство запросов шло через Redis, поэтому БД не была главным лимитом".

## 113. Что ограничивало RPS в материалах?

По load test отчетам:

- Redis давал быстрый read path;
- Oracle pool ограничивал write/fallback сценарии;
- nginx backlog стал потолком на большом concurrency;
- CPU не был главным лимитом;
- при Redis failover трафик пошел бы в БД и потолок резко снизился бы.

## 114. Как объяснить p95 и p99?

p95 = 95% запросов быстрее этого значения, 5% медленнее. p99 показывает хвост latency и часто важнее среднего.

Среднее может быть хорошим, хотя часть пользователей получает timeout.

## 115. Как выбирать размер connection pool?

Пул должен соответствовать:

- лимиту БД;
- средней длительности запроса;
- числу replicas;
- профилю нагрузки;
- latency SLO.

Слишком маленький pool дает очереди. Слишком большой перегружает БД и увеличивает context switching.

## 116. Что такое thundering herd?

Thundering herd — много запросов одновременно видят cache miss и все идут в source of truth.

Решения:

- Redis lock `SET NX`;
- request coalescing;
- stale-while-revalidate;
- preload cache;
- random TTL jitter.

## 117. Как не потерять MDC/correlation id в reactive?

ThreadLocal MDC плохо работает при переключении потоков. Нужно использовать Reactor Context или Micrometer Context Propagation. Если MDC ставится вручную, его нужно очищать в `doFinally`.

Логи без correlation id бесполезны при production incident.

## 118. Что такое MapStruct и почему он полезен?

MapStruct генерирует mapper-код на compile-time. Это быстрее reflection-based mapping и ловит ошибки маппинга на этапе сборки.

Минус: нужно следить за явностью mapping'ов и не прятать сложную бизнес-логику в mapper.

## 119. Как отвечать на вопрос "что бы улучшили в архитектуре"?

Хороший ответ должен быть конкретным:

- убрать blocking MinIO с event loop;
- включить circuit breaker на внешних API;
- добавить cache stampede protection;
- ограничить concurrency в batch processing;
- добавить DLQ для Kafka;
- добавить вторую replica для HA;
- покрыть критичные сценарии load tests.

## 120. Какие вопросы задать работодателю?

Спросить:

- какая бизнес-задача у продукта;
- монолит, модульный монолит или микросервисы;
- версия Java и Spring;
- какие главные production-боли;
- как устроены code review и CI/CD;
- есть ли on-call;
- какие SLO/SLA;
- чего ждут от senior в первые 1-3 месяца.

