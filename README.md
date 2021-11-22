# highload-spotify

## Тема работы

Проектирование высоконагруженной системы - аналога Spotify.

## Основная функциональность

- Прослушивание музыки
- Получение информации о композициях, альбомах и исполнителях

## Целевая аудитория[1]

- Месячная аудитория: 365 млн. человек.
- Дневная аудитория: 44% от MAU = 160.6 млн. человек[2]
- Пользователей с платной подпиской: 165 млн. человек.
- Распределение аудитории по странам:

_Таблица 1_

Европа  | Северная Америка  | Латинская Америка  | Остальной мир  |
--------| ------------------| -------------------| ---------------|
121 млн.|       85 млн.     |       78 млн.      |     71 млн.    |

- Распределение аудитории по возрасту:

_Таблица 2_

18-24  | 25-34  | 35-44  | 45-54  |   55+   |
-------| -------| -------| -------| --------|
26%   |  29%   |  16%   |  11%   |   19%   |

## Расчет нагрузки

### Объем хранилища

Рассчитаем размер одной песни.

Битрейт композиций для бесплатной и платной версий отличается - 128 Кбит/с и 320 Кбит/с соответственно.[3]

Длительность одной композиции составляет в среднем от 3 до 5 минут.[4] Для удобства расчетов возьмем среднее значение в
4 минуты. Тогда средний размер одной песни:

```4 * 60 * 128 = 30720 Кбит = 3.75 Мб``` - в бесплатной версии

```4 * 60 * 320 = 75800 Кбит ~ 9.4 Мб``` - с подпиской

Если учесть долю пользователей с подпиской (```165/365 ~ 0.45```), то в среднем размер одной композиции:

```3.75 * 0.55 + 9.4 * 0.45 ~ 6.3 Мб ```

Количество композиций в Spotify - более 70 млн.[5] Каждую песню необходимо хранить в двух битрейтах, поэтому минимальный
объем хранилища для всех композиций:

```70 * 10^6 * (3.75 + 9.4) ~ 878 Тб```

### Сетевая нагрузка

DAU (указано выше): 160.6 млн. человек. В среднем пользователи Spotify тратят около 118 минут в день на прослушивание
музыки.[6] Вес одной минуты средней композиции:

```6.3 / 4 ~ 1.6 Мб```

В таком случае дневной трафик равен:

```160.6 * 10^6 * 118 * 1.6 / 1024 / 1024 ~ 28917 Тб```

Необходимая пропускная способность:

```160.6 * 10^6 * 118 * 1.6 / 1024 / 1024 * 8 / 24 / 60 / 60 ~= 2.68 Тбит/с```

Также необходимо учесть, что дневной трафик меняется в течение дня и в пиковые часы и выходные дни может быть до двух
раз выше.[7]
Поэтому в дальнейших расчетах будем использовать значение в 5.5 Тбит/с.

### RPS

Среднее время ежедневного использования сервиса[1]:

```(110+99+117+124+140) / 5 = 118 минут```

За 118 минут пользователь Spotify может прослушать до 30 композиций. В процессе прослушивания отдельно взятой песни
совершаются запрос за самим файлом, а также запрос за информацией о песне.

Рассмотрим типовой ежедневный пользовательский сценарий, состоящий из:

- Авторизация
- Выбор плейлиста и его загрузка
    - Загрузка данных пользователя
    - Загрузка главной страницы со списком плейлистов
    - Загрузка конкретного плейлиста
- Смена плейлиста
    - Поиск нового плейлиста (возвращает список подходящих плейлистов)
    - Загрузка конкретного плейлиста
- Переключение на следующий трек
- Прослушивание трека
    - Получение аудиофайла
    - Загрузка информации о треке (название, исполнитель) и картинки трека

Чтобы численно посчитать RPS, примем, что действия 2 и 3 совершаются средним пользователем по 5 раз за сессию
прослушивания в 118 минут.

Предполагаемая нагрузка:
```160.6 * 10^6 * (1 + 2 * 30 + 3 + 5 * 2 + 5) / 24 / 60 / 60 ~= 146845 RPS```

Посчитаем RPS отдельно для каждой группы действий:

1. Авторизация

```160.6 * 10^6 * 1 / 24 / 60 / 60 ~= 1859 RPS```

2. Загрузка списка плейлистов:

```160.6 * 10^6 * 5 / 24 / 60 / 60 ~= 9294 RPS```

3. Загрузка плейлиста:

```160.6 * 10^6 * 5 / 24 / 60 / 60 ~= 9294 RPS```

4. Загрузка трека.

Загрузка трека включает в себя запрос за треком и за информацией о нем (исполнитель, картинка). То есть необходимо
сделать 30 * 2 запросов:

```160.6 * 10^6 * 2 * 30 / 24 / 60 / 60 ~= 111628 RPS```

5. Поиск

Поиск выполняется аналогично с загрузкой списка плейлистов:

```160.6 * 10^6 * 5 / 24 / 60 / 60 ~= 9294 RPS```

Сводка по RPS:

_Таблица 3_

Авторизация | Загрузка списка плейлистов  | Загрузка плейлиста | Загрузка трека  | Поиск
-------| -------| -------| -------| -------|
1859 RPS | 9294 RPS   |  9294 RPS   |  111628 RPS   |  9294 RPS

## Логическая схема базы данных

![Логическая схема БД](db_logical.png)

## Физическая схема базы данных

Произведем денормализацию: в таблицы tracks, tracks_queue, tracks_history добавим дополнительные поля.

![Физическая схема БД](db_physical.png)

### Redis

Информацию о сессиях и очередях будем хранить в Redis. Redis хранит данные в памяти, что обеспечивает быстрое обращение
к нужной информации.

Для таблицы очередей для каждого пользователя будем хранить текущий альбом и текущую песню, тогда ежедневно необходимо:
``4(id) + 4(track_id) + 256(track_name) + 256(artist_name) + 256(album_name) + 4(user_id) + 4(album_id) = 784 байт``
``784 * 160.6 * 10^6 ~= 117 Гб``

Сессии пользователя: ``3 * 16 * 160.6 * 10^6 = 7 Гб``

В целом необходимо: ``117 Гб + 7 Гб = 124 Гб``.

### PostgreSQL

Основные данные сервиса будут храниться в PostgreSQL. Выполним оценку объема данных для хранения.

Пользователь:
``4(id) + 256(name) + 256(email) + 512(pwdhash) + 1(is_premium) + 4(history_id) + 4(queue_id) = 1037 байт``

Трек:
``4(id) + 256(name) + 4(duration) + 4(id_artist) + 2*256(path) + 256(artist_name) + 256(album_name) = 1292 байт``  
Всего 70 млн треков

Альбом: ``4(id) + 256(name) + 4(duration) + 4(tracks_number) + 4(artist_id) = 272 байт``  
Среднее количество песен в музыкальном альбоме 12-14[8], возьмем среднее - 13, тогда у нас будет примерно 5.4 млн
альбомов.

Исполнитель: ``4(id) + 256(name) + 1024(description) = 1284 байт``  
Пусть у каждого исполнителя будет по 5 альбомов, тогда у нас всего 1.1 млн исполнителей.

История прослушивания. Будем хранить треки, прослушанные пользователем за последние сутки.
``4(id) + 4(track_id) + 4(user_id) + 4(album_id) + 256(track_name) + 256(artist_name) + 256(album_name) + 4(date) = 788 байт``
``788 * 30 * 160.6 * 10^6 = 3.5 Тб``

Связь многие ко многим: ``3 * 4(id) = 12 байт``

Итого: ``(1037 * 160.6 (пользователи) + 1292 * 70 (треки) + 272 * 5.4 (альбомы) + 1284 * 1.1 (исполнители) + 788 * 30 * 160.6 (история) + 12 * (70 + 1.1 + 70)) * 10^6 = 285196600000 байт = 3.7 Тб``

Также для основной базы поставим две реплики, которые будут работать в роли Slave по отношению к основной базе (Master).

После денормализации мы можем легко шардировать таблицы tracks_queue и tracks_history.

## Аудиофайлы

Для аудиофайлов понадобятся дополнительные сервера, распределенные по миру. Были выбраны следующие города:

- Франкфурт (Германия)
- Лондон (UK)
- Los Angeles (US)
- Washington D.C. (US)
- New York (US)
- Москва (Россия)
- Владивосток (Россия)
- Дели (Индия)
- Бразилиа (Бразилия)
- Мехико (Мексика)
- Сидней (Австралия)

По ним будет распределяться нагрузка в 5.5 Тбит/сек с учетом информации из Таблицы 1. Кроме этого нам необходимо хранить
878 Тб медиатеки, для этого пусть на серверах будут SSD диски объемом 3,84 Тб по 26 штук на каждый. Тогда один сервер
может хранить 99,84 Тб, то есть для целой медиатеки нужно 9 серверов.

Для обеспечения отказоустойчивости объединим диски в RAID 5 массив (то есть понадобится 26+1 дисков). Также в каждом
городе арендуем 2 дата-центра, в котором будут храниться основные и резервные сервера. Итого 18 серверов на один город.

_Таблица 4_

| Город       | Часть  | Нагрузка (Тбит/с) | Кол-во серверов
|-------------|--------|-------------------|--------------
|Франкфурт    |0.113   | 0.62              | 18
|Лондон       |0.113   | 0.62              | 18
|Москва       |0.113   | 0.62              | 18
|Los-Angeles  |0.08    | 0.44              | 18
|Washington   |0.08    | 0.44              | 18
|New York     |0.08    | 0.44              | 18
|Владивосток  |0.067   | 0.37              | 18
|Бразилиа     |0.11    | 0.61              | 18
|Мехико       |0.11    | 0.61              | 18
|Дели         |0.067   | 0.37              | 18
|Сидней       |0.067   | 0.37              | 18

Запросы на эти кластеры будут приходить через DNS, но каждому понадобится свой балансировщик для распределения запросов
между серверами. Каждому балансировщику дадим по одной реплике для обеспечения отказоустойчивости.

## Картинки

К каждому треку идет картинка - обложка песни. С помощью инструментов разработчика было выяснено, что средний размер
обложки в веб-версии Spotify составляет около 40 kB. Для 70 млн. композиций общий размер обложек:

```70 * 10^6 * 40 / 1024 / 1024 / 1024 ~= 2.7 Тб```

Для хранения всех картинок хватит одного SSD диска на 3,84 Тб. Разместим по одному серверу с RAID 1 массивом (2 диска) в
каждом из двух наших дата-центров. Итого на каждый город придется 4 сервера, отдающих картинки.

Итоговое количество серверов в каждом городе: 18 + 4 = 22.

## Сетевое оборудование

Наибольшая сетевая нагрузка приходится на европейские сервера (0.62 Тбит/с = 634.88 Гбит/с). 
Это значит, что на каждый из 18 серверов аудио приходится по ```634.88 / 18 ~= 35.3 Гбит/с```.
На каждом сервере объединим в Bonding 4 оптических сетевых интерфейса с пропускной способностью 10 Гбит/с, получив итоговую пропускную способность сервера в 40 Гбит/с.

## Выбор технологий

Для написания бэкенда используем язык Golang как язык, который хорошо зарекомендовал себя в разработке высоконагруженных
приложений и на который несложно найти специалистов. В случае использования микросервисной архитектуры (например,
микросервисы авторизации и прослушивания альбома) можно использовать систему gRPC. Для написания фронтенда используем
TypeScript, для создания SPA-приложения и более быстрой работы с DOM будет использована библиотека React. В качестве
балансировщика и реверс-прокси будет использован Nginx.

## Схема проекта

Подведем итог всех расчетов и отобразим результат в виде схемы:

![Схема проекта](project_schema.png)

| Сервер    | CPU  | RAM | Диск                          | Кол-во   | Местоположение
|-----------|------|-----|-------                        |-------   |---
|Бэкенд     |32    | 256 | 32Гб х 1SSD                   | (1+1)*11 |Города из _Таблицы 4_
|PostgreSQL |32    | 512 | 256Гб х 1SSD                  | (1+2)*11 |Города из _Таблицы 4_
|Redis      |16    | 128 | 32Гб х 1SSD                   | (1+2)*11 |Города из _Таблицы 4_
|Аудио      |32    | 256 | 3.84Тб х 27SSD | 11*18 |Города из _Таблицы 4_
|Картинки      |32    | 256 | 3.84Тб х 2SSD | 11*4 |Города из _Таблицы 4_
|Nginx      |32    | 256 | 64Гб х 1SSD                   | (1+1)*11 (аудио) + (1+1)*11 (картинки) + (1+1)*11 (проксирование к бэкенду)|Города из _Таблицы 4_

## Список источников

[1] https://www.businessofapps.com/data/spotify-statistics/

[2] https://www.goodwatercap.com/thesis/understanding-spotify

[3] https://support.spotify.com/us/article/audio-quality/

[4] https://www.vox.com/2014/8/18/6003271/why-are-songs-3-minutes-long

[5] https://newsroom.spotify.com/company-info/

[6] https://www.gwi.com/reports/music-streaming-around-the-world

[7] https://habr.com/ru/company/dcmiran/blog/496542/

[8] https://ru.wikipedia.org/wiki/%D0%9C%D1%83%D0%B7%D1%8B%D0%BA%D0%B0%D0%BB%D1%8C%D0%BD%D1%8B%D0%B9_%D0%B0%D0%BB%D1%8C%D0%B1%D0%BE%D0%BC
