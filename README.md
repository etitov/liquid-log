# Liquid log

Практическое задание для курса ИМКН "Промышленая разработка на Java (2017)"

## История для практического задания
Вы работаете в компании, которая разрабатывает корпоративные приложения. Они установлены у сотен клиентов.
Чаще всего доступ к приложению возможен только из локальной сети клиента.

Коллеги из соседнего отдела решили оценить производительность своего приложения, установленного у конкретных клиентов.
Задача не совсем в мониторинге производительности, а в ретроспективном анализе - как обычно приложение работает? как оно работало вчера?
Менять инфрастуктуру клиента возможности нет, поэтому единственная доступная для анализа информация - в логах приложения.

В результате, для своих нужд коллеги и написали это приложение.

Вам понравился их опыт, но вот незадача - вы разрабатываете совсем другое приложение, которое пишет ~~неправильные~~ совсем
другие логи. Поэтому вы решили доработать это приложение таким образом, чтобы оно могло парсить _почти_ любые логи.

Но для начала придётся разобраться - как именно устроено приложение внутри.

## Что такое Liquid log

Liquid log - отображение производительности какого-либо приложения на основе логов этого приложения.

Оцениваемое приложение является Web-приложением, написанным с использованием GWT. Клиентская часть приложения (из web-браузера)
отправляет на сервер команды о выполнении каких-то действий - Action-ы. Действие выполняется на сервере, и его
результат отправляется обратно клиенту. Таким образом, отзывчивость приложения - это скорость выполнения action-ов.

Запись в лог при выполнении действия может выглядеть так:
```
20156544 [AddObjectAction(ntsclass$ntstype) naumen #303 192.168.112.104 naumen] (07 сен 2017 10:34:03,179) INFO  dispatch.Dispatch - SQL(70) Done(296):AddObjectAction [getFqn()=ntsclass$ntstype, getActionDebugTokens()=[ntsclass$ntstype], getObject()=SMapProperties: { parent = root$401, string = Общий отдел, requestDate = Thu Sep 07 10:33:54 YEKT 2017, title = 1, ouLink = SimpleTreeDtObject [ou$5806,], parentUUID = root$401 }, ]
```

Это запись о добавлении объекта (отдела). Важные для нас части этого лога:
* `AddObjectAction` - название действия
* `07 сен 2017 10:34:03,179` - дата-время выполнения действия
* `SQL(70)` - количество миллисекунд, потраченных на выполнение SQL в данном действии
* `Done(296)` - количество миллисекунд, потраченных на выполнение действия на сервере в целом

Приложение запускается в двух режимах - (1) парсинг лога (и сохранение его в БД) и (2) отображение лога.

## Тестовые данные

В корне проекта лежит архив `sdng.log.2017-09-07.zip`. Его нужно распаковать и положить файл в любую пользовательскую директорию.
Файл, который лежит в архиве - это лог реального приложения (но на тестовом стенде). Этот лог содержит достаточное количество данных
для того, чтобы начать работу и проверить парсинг и отображение графиков.

Для удобства, все команды в последующих примерах используют имя именно этого файла.

## Запуск парсинга лога

### В IDEA
1. VM Options: `-DParser -Dparse.mode=sdng -Dinflux.host="http://127.0.0.1:8086" -Dinflux.user="root" -Dinflux.password="root"`
1. Аргументы: `"/home/user/sdng.log.2017-09-07" sdng`

Первый аргумент - это путь до файла с логом, который будет парситься.
Второй аргумент - это имя базы данных, в которую будет складываться результат парсинга. В данном случае это `sdng`.

Доступно 3 режима:
* sdng - для логов приложения ServiceDesk
* gc - для логов Garbage Collector
* top - для логов утилиты top

Для парсинга и последующего отображения необходима база данных [InfluxDB](https://github.com/influxdata/influxdb)

Для всех студентов будет доступна InfluxDB, развёрнутая в OpenShift. В процессе разработки можно использовать эту базу, 
но можно также настроить InfluxDB на локальной машине.
Доступ к базе Influx прописан в файле конфигурации application.properties

### Из командной строки

* Перейти в рабочую директорию приложения
* Собрать приложение командой `mvn clean install`
* В результате должна быть запись `BUILD SUCCESS`, а также должен получиться WAR-файл: `Building war: ${some_directory}/liquid_log/target/liquid_log-0.0.1-SNAPSHOT.war`
* `java -DParser -Dparse.mode=sdng -Dinflux.host="http://127.0.0.1:8086" -Dinflux.user="root" -Dinflux.password="root" -jar target/liquid_log-0.0.1-SNAPSHOT.war "/home/user/sdng.log.2017-09-07" sdng`
* Обратите внимание, что выполнение происходит из корня проекта - это нужно для того, чтобы приложение нашло папку config

Описание аргументов см. в разделе запуска в IDEA.

## Запуск web-приложения

### Локально

Настройки доступа до InfluxDB указаны в файле application.properties. По умолчанию приложение сконфигурировано
на запуск из Openshift - используются заданные имена переменных окружения.

Чтобы запускать на локальном компьютере, следует:
* в корне приложения создать папку config
* скопировать в эту папку файл application.properties
* поменять значения доступа к InfluxDB на нужные

При запуске из рабочей директории проекта, будет использоваться этот локальный файл настроек.

Папка config добавлена в gitignore, поэтому не будет отправляться вместе с другими изменениями в репозиторий.

#### Из IDEA
Из IDEA нужно только запустить Main class: ru.naumen.perfhouse.PerfhouseApplication
Без указания дополнительных опций виртуальной машины и без всяких аргументов.

#### Из командной строки

* Перейти в рабочую директорию приложения
* Собрать приложение командой `mvn clean install`
* В результате должна быть запись `BUILD SUCCESS`, а также должен получиться WAR-файл: `Building war: ${some_directory}/liquid_log/target/liquid_log-0.0.1-SNAPSHOT.war`
* Выполнить команду: `java -jar target/liquid_log-0.0.1-SNAPSHOT.war`
* Обратите внимание, что выполнение происходит из корня проекта - это нужно для того, чтобы приложение нашло папку config
* Если всё прошло удачно, последнее сообщение в логе будет: `Started PerfhouseApplication in N.nnn seconds`
* Можно открыть в браузере [http://localhost:8080/](http://localhost:8080/) и проверить доступность приложения

### В Openshift
см. лекцию.

## Как проверить данные
После того, как парсинг прошёл успешно, можно запустить веб-приложение и проверить данные (или зайти в веб-приложение в openshift).

Приложение по умолчанию имеет заданные выборки. Но данный файл лога для этих выборок устарел, поэтому нужно:
* На стартовой странице выбрать базу sdng
* Нажать Custom request
* Выбрать в качестве начала периода - 2016 год.
* Нажать Request

Должен отобразиться график.