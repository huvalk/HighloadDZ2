## Тема
Сервис "Всероссийский электронный дневник школьника"
## Определение возможного диапазона нагрузок 
МЭШ - Московская электронная школа расчитана только на жителей Москвы. По данным [росстата](https://rosstat.gov.ru/bgd/regl/b20_111/Main.htm) 1 млн детей и подростков в возрасте от 7 до 18 лет учаться в московских школах. 
По данным на [2016 год](https://www.mos.ru/news/item/15697073/?utm_source=search&utm_term=serp) из 169 московских школ, в которых была введена система "Электронный дневник", количество пользователей-кольников составляет 400 тыс. человек. А количество уникальных пользователей в месяц - 1 млн.
Исходя из данных о [средней наполняемости школ](https://medportal.ru/enc/parentschildren/school/16/#:~:text=%D0%A7%D0%B0%D1%81%D1%82%D0%BD%D1%8B%D1%85%20%D1%88%D0%BA%D0%BE%D0%BB%20%D0%B2%20%D0%A0%D0%BE%D1%81%D1%81%D0%B8%D0%B8%20%D0%BD%D0%B0%D1%81%D1%87%D0%B8%D1%82%D1%8B%D0%B2%D0%B0%D0%B5%D1%82%D1%81%D1%8F,%D1%87%D0%B0%D1%81%D1%82%D0%BD%D0%BE%D0%B9%20%E2%80%93%20%D0%BE%D1%82%20100%20%D0%B4%D0%BE%20200.) в 950 человек, можно прийти к выводу, что сервисом эжедневно пользовался практически каждый школьник.

В среднем у учащегося 6 предметов в день, это означает, что для выполнения таких действий как: просмотр материалов по курсу, просмотр домашних заданий, просмотр оценок, загрузка домашних заданий, потребуется `6 * 4 = 24` запросов в день.

В среднем у учителя 6 уроков в день в классах по 18 человек (согласно [статистике росстата](https://www.mk.ru/social/2020/09/30/kolichestvo-shkolnykh-uchiteley-rezko-sokrashhaetsya-nashli-udobnuyu-alternativu.html)) это означает, что для выполнения таких действий как: проверка домашних заданий, выставление оценок, отслеживание посещаемости требуется `6 * 18 * 3 = 324` запроса в день.

Итого `(1 000 000 * 24 + 324 * 1 000 000 / 18) / (24 * 60 * 60) = 486` rps - нагрузка на Московский электронный дневник.

## Выбор планируемой нагрузки
Сервис ориентирован на российскую аудиторию. По данным [росстата](https://rosstat.gov.ru/bgd/regl/b20_111/Main.htm) 24.8 млн детей и подростков в возрасте от 7 до 18 лет. Учитывая [проникновение интернета в России на 2020 год](https://www.web-canape.ru/business/internet-2020-globalnaya-statistika-i-trendy/) в 81%, получаем: `24.8 * 81% = 19.8` пользователей-школьников по России.
Во время противовирусного карантина все школы вынуждены использовать электронные дневники. В месяц к пользователям добавляются родители, ежемесечно проверяющие результаты учебы школьников. Ориентируясь на соотновешние учеников / родителей в Москве - `400 / 550`, учитывая соотношение школьников в Москве / России - `1 / 19.8`, получаем:
- `19.8 + 550 / 400 * 19.8 = 47` млн. пользователей в месяц.
- `468 * 19.8 = 9266` - среднее значение rps

Учитывая, что [40 млн населения России](https://superresearch.ru/time-zones), то есть больше четверти, проживает в московском часовом поясе, можно сделать вывод, что пиковая нагрузка за 8 часовой рабочий день составляет:
`(19.8 / 4) * (8 / 24) * 9266 = 5 / 3 * 9266 = 15444` rps


## Логическая схема базы данных
Выделенные сущности: 
- пользователь, 
- преподаватель, 
- ученик, 
- класс, номер класса определяется годом начала обучения параллели, буква класса хранится отдельно, 
- контрольная точка, домашняя или контрольная работа или другое мероприятие,
- предмет, представлен названием предмета, позволяет реализовать ситуацию, в которой один преподаватель ведет много предметов,
- ответ студента на задание содержит, помимо оценки и самого ответа, комментарии к заданию, хранимые строкой в денормализованном виде, что позволит сэкономить на операции join между крупными таблицами самих отевтов и комментариев, приведя отношения "многие комментарии к одному заданию" в отношения один-к-одному. Предполагается, что файл с работой ученика будет храниться отдельно, а сущность будет только содержать путь к нему.

![Логическая схема](src/logic_scheme.png)

## Физическая система хранения
Исходя из модели предметной области, можно сделать наблюдения:
1) От сущности пользователя не зависят остальные, что позволяет создать на ее основе отдельный сервис авторизации.
2) Такие сущности как: преподаватель, ученик, класс и контрольная точка практически не подвержены изменениям и создаются единыжды перед началом учебного года. Это означает, что основная операция, выполняемая над ними - чтение. 
3) Сущность ответа ученика потенциальна подвергнута частным изменениям.

####Рассмотрим каждый пункт отдельно:
####1
Реализуя сессия пользователя по технологии JWT, можно избавиться от необходимости верифицировать сессию пользователя, проверяя ее id в базе данных. Таким образом пользователю достаточно единыжды авторизоваться. В шардировании данной базы нет необходимости, достаточно репликацию master-slave, так как основная нагрузка будет на чтение.
Предположим, что авторизация выполняется раз в месяц, тогда при количестве пользователей в месяц `= 47` млн, получим средний qps

`47 000 000 / (30 * 24 * 60 * 60) = 18` qps

Предположим случай пиковой нагрузки, например, 1 сентября, когда все пользователи будут авторизовываться, получим средний qps:

`47 000 000 / (24 * 60 * 60) = 5400` qps

И `54000` qps при 10-ти кратном запасе.
Расчитаем занимаемое одной записью место:

| Поле          |  Память (байт)     |
| ------------- |------------------| 
|id |8 |
|login | ~80|
|password | ~80|
|student | 8|
|teacher | 8|
|**Суммарно** | **194**|

Следовательно на 47 млн активных пользователей понадобится 8,5 Гб.
С учетом того, что ежегодный прирост пользователей `= 47 / 11 = 4.3` млн, минимального объема памяти будет достаточно.
Под задачи хранения данных подойдет noSQL база данных **Redis** - хранящая базу в оперативной память в формате ключ-значение, переодически выполняя операции записи на диск. Потенциально Redis может выдержать многократные (10-ти кратные) пиковые нагрузки в [50 тыс. rps](https://skipperkongen.dk/2013/08/27/how-many-requests-per-second-can-i-get-out-of-redis/).
####2
 Большое количество связей, в том числе связи много-ко-многим, и то, что изменения данных практически не производится, поддталкивает к использованию SQL базы данных. Кроме того, есть возможность шардирование студентов по годам, что позволит распределить нагрузку на разные машины. Поэтому выбор сделан в сторону **PostgreSQL**. 
[Согласно бенчмаркам](https://habr.com/ru/company/drtariff/blog/383295/)  ssd-диски позволят добится 20 тыс. qps с одной машины, что дает 2-х кратный запас к rps, определенный в пункте "Выбор планируемой нагрузки".
####3
В среднем, ученик изучает 5 предметов в день по расписанию, а значит может оставлять ответы на около 5 заданий в день и 5 исправлений, преподаватель оставлять 1 комментарий и дважды проверять работу. Что в сумме дает 10 put запросов ответа на задание от ученика, 10 get запросов ответа на задание от преподавателя. Само задание, хранимое в pdf, может занимать около 5 Мб. Что суммарно дает:

`5 * 20 000 000 * 5 = 500` Гб в день, что за 125 учебных дней в полугодии даст `65 000` Тб данных для хранения. Ученические работы хранятся в течении полугодий, поэтому ежегодный прирост не ожидается.

Для обработки такого количество файлов хорошо подходит s3 хранилище, которое позволит снять с сервиса нагрузку на получение/хранение ответа на задание. Как один из вариантов - **Mail S3**

Расчитаем занимаемое на диске место одной записью в таблице:
| Поле          |  Память (байт)     |
| ------------- |------------------| 
|student |8 |
|check_point | 8|
|mark | 8|
|answer | [~200](https://gpdb.docs.pivotal.io/510/admin_guide/external/g-s3-protocol.html)|
|comment | ~1000|
|**Суммарно** | **1224**|

Итого около 1Кб.
Комментарии к ответу храняться в строке, что позволяет исключить необходимость делать join при выборке комментариев к конкретному ответу.
Расчитаем возможный tps, исходя из предположения, что учитель и ученик создают 20 запросов в день для проверки задания, исправления, комментирования:

`(20 * 20 000 000) / (24 * 60 * 60) = 4500` tps.

Исходя из этого, выбор пал на **MongoDB**, которая в отличие от Redis поддерживает вторичные ключи и смешанные индексы, поддерживает репликацию стандартными инструментами и обладает высокими показателями [доступности](http://www.mongodb.org/display/DOCS/Replica+Sets). В силу того, что данная база будет иметь дело с большим количеством данных, имеет смысл настроить шардинг. Как и в случае с postgres шардинг можно выполнить по времени, например по учебным полугодиям. Сами данные, в сочетании с S3 хранилищем помещаются в  16 Мб ограничение.

## Выбор прочих технологий
Для написания бэкэнда выбран Golang, в его пользу говорят следующие плюсы:
- высокая скорость разработки,
- производительность, [сравнимая](https://benchmarksgame-team.pages.debian.net/benchmarksgame/fastest/go-gpp.html) с c++,
- автоматической управление потоками позволяет добится равномерного распределения нагрузки на ядра процессора,
- удобная система модулей.

Для исключения лишнего уровня абстракций и связанного с ним накладными расходами, связь с СУБД будет производится напрямую через ее драйвер, не используя ORM библиотек.
Данные между клиентом и сервером будут передаваться в формате JSON. Для сериализации и десериализации JSON будет использоваться библиотека easyJSON, заменяющая механизм рефлексии, на кодогенерируемые функции.
В качестве протокола межсервисного взаимодействия будет зайдествован gRPC.
Для клиент-серверного взаимодействия задействован HTTP/2, за счет улучшенных алгоритмов сжатия http пакетов и возможности мультиплексирования запросов в одном соединении.

## Расчет нагрузки и потребного оборудования
**Микросервисы** 
Разделение на микросервисы соответствует разделению в разделе "Физическая система хранения", то есть:
1) авторизация/регистрация,
2) расписание, информация об учетелях, учениках и контрольных мероприятиях,
3) контрольные работы учеников

<a name="1"><h4>1</h4></a> 
Как уже было расчитано, для хранения пользователей не потребуется много места, однако, так как каждый запрос на сервер будет соправаждаться верификацией сессии, нагрузка на данную часть сервиса будет максимальной, по сравнению с остальными частями, в целях запаса стоит ориентироваться на двукратное пиковое значение, то есть на `30 000` rps. Объем памяти с запасом на 12 лет:
`8 Гб + 12 * 1 Гб = 20 Гб`
<a name="2"><h4>2</h4></a>  
Проведем расчеты требуемого объема памяти.
| Таблица          |  Память (байт)     | Количество записей на пользователя
| ------------- |------------------|------------------| 
|Students | 132|пользователь или учитель, или ученик
|Teacher | 150|пользователь или учитель, или ученик
|Class | 24| 0.05
|Schedule | 52| 2
|Subject | 96| const = 30
|CheckPoint | 1032| 32

Итого суммарный объем хранимых данных за год: `630` Гб
С учетом того, что прирост пользователей фиксированный и составляет примерно 10% и тем, что такие данные как расписание и контрольные точки потенциально могут меняться каждый год, имеем: `3.5` Тб с запасом на 5 лет.
<a name="3"><h4>3</h4></a>  
Для хранения файлов работ подойдет тариф S3 Standard – Infrequent Access, обеспечивающий доступ к файлам за миллисекунды.
Как было расчитано ранее, 1 запись в таблице StudentWork занимает около `1224` байт. Количество записей в таблице на одного пользователя `= 640`.
А значит, требуемый объем хранилища: `1224 * 640 = 14.5` Тб

Следует учесть, что ежегодный прирост данных 100%, а значит с запасом на 5 лет потребуется `71.5` Тб.

Сделаем предположения, какое количество RPS сможет обеспечить одно ядро для каждой задачи и исходя из этого, посчитаем необходимое количество ядер. Разделим условно запросы на медленные - 100 rps, и быстрые - 500 rps.
- Авторизация - 500 rps
- Получить расписание - 100 rps
- Получить список класса - 100 rps
- Получить оценки ученика - 100 rps
- Загружать задание - 500 rps
- Оставить комментарий - 500 rps
- Оценить задание - 500 rps

Исходя из этого, расчитаем средний rps, с которым надо справится одному ядру каждого микросервиса и исходя из пиковых значений rps, оценим, сколько понадобится машин и ядер.
Для расчета серверов для БД помимо этого учитывалось количество шардов.

Пиковый rps на узлы бэкэнда с двукратным запасом:

Узел|rps
---|---
[Авторизация/регистрация](#1)| 30 000
[Расписание](#2)| 30 000
[Сданные работы](#3)| 10 000

Требуемое оборудование:
Узел|Cores|RAM| Количество
---|---|---|---
[БД 1](#1)|16 |128 | 1 master, 2 slave = 3
[БД 2](#2)|16 |128 | 11 master, 2*11 slave = 33
[БД 3](#3)|32 |128 | 11 master, 2*11 slave = 33
[Авторизация/регистрация](#1)|32 |64 |8
[Расписание](#2)|32 |128 |8
[Сданные работы](#3)|32 |128 |4
Фронтенд|32 |64 |4

Расчет нагрузки на фронтенд выполнялся следующим образом: пусть фронт весит 5 Мб, тогда при пиковой нагрузке в 15 тыс. rps, с двухкратным запасом, получим трафик в 150 Гб/сек. Два сервера со 100GbE способны выдержать такую нагрузку, но возьмем запас, на случай роста или на случай неполадок с сетью.

Трафик на бекенд серверах в пике не привышает 1 Гб/сек. и не является бутылочным горлышком для системы.
<a name="disk"><h4>Расчет дисков</h4></a>  
В каждом сервере жесткие диски SSD.
Произведем расчет жестких дисков баз данных:

Узел|Эффективный объем (Гб)| Объем на одном шарде | Диски| Модель| Стоимость (руб./шт.)
---|---|---|---|---|---
[БД 1](#1)| 64|64| 2*2 диска по 32 Гб в RAID10|[7N47A00129]| 6 000
[БД 2](#2)| 4096|400| 2*2 диска по 200 Гб в RAID10|[400-AWHC]| 11 000
[БД 3](#3)| 82 000|8 200| 2*4 диска по 1 Тб в RAID10|[400-AJQD]| 21 500

[7N47A00129]: https://www.onlinetrade.ru/catalogue/servernye_zhestkie_diski_i_ssd-c4239/lenovo/nakopitel_ssd_lenovo_m.2_1x32gb_sata_7n47a00129-1413727.html
[400-AWHC]: https://www.onlinetrade.ru/catalogue/servernye_zhestkie_diski_i_ssd-c4239/dell/nakopitel_ssd_dell_2.5_1x240gb_sata_hot_swapp_400_awhc-2014001.html
[400-AJQD]: https://www.onlinetrade.ru/catalogue/servernye_zhestkie_diski_i_ssd-c4239/dell/zhestkiy_disk_dell_2.5_1x1.2tb_sas_10k_hot_swapp_400_ajqd-2066353.html 
## Выбор хостинга и расположения серверов
Для хостинга выбрано cloud.croc.ru. Расположение серверов ограничено расположением датацентров крока в России - только Москва.

Для хранения файлов работ подойдет тариф в S3 подойдет хранилище icebox.

## Схема балансировки нагрузки 
В качестве балансера будет использоваться nginx. Так как предполагается использование протокола HTTP/2, выбрана L7 балансировка. Терминацию SSL выполняет nginx. Балансировка на уровне DNS выполняется алгоритмом Round Robin с ручной уборкой неотвечающих ip адресов.

## Обеспечение отказоустойчивости
Для обеспечения отказоустойчивости каждый сервер БД имеет master-slave репликацию, по 2 слэйва на один мастер. 

Для мониторинга системы будут использованы Grafana + Prometeus.

## Расчет стоимости
Крок предоставляет возможность располагать в датацентре сервера, поэтому расчет будет вестись не стоимости виртуальных машин, а реальных серверов.
#### S3
![стоимость s3](src/s3_price.png)

#### Сервера
Стоимость дисков перечислена [выше](#disk). В итоге общая стоимость дисков выйдет в `7.2` млн. руб.

Расчитаем стоимость серверов:
Выбрана платформа [PowerEdge R6515]. Конкретные цены получены после конфигурации по ссылке.
Cores|RAM|Стоимость (руб./шт.)| Кол-во
---|---|---|---
16|128| 320 000| 36
32|64| 320 000| 12
32|128| 400 000| 45

[PowerEdge R6515]: https://www.dell.com/en-us/work/shop/servers-storage-and-networking/poweredge-r6515-rack-server/spd/poweredge-r6515/pe_r6515_13732a_vi_vp
Итого, общая стоимость серверов составит `33.5` млн. руб.

#### Итого
Железо будет стоить `41` млн. руб. 
Обслуживание S3 в год - `4.6` млн. руб.

