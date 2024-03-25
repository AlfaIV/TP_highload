# Проектирование высоконагруженного пространства для командной работы
Курсовая работа в рамках 3-го семестра программы по Веб-разработке ОЦ VK x МГТУ им. Н.Э. Баумана (ex. "Технопарк") по дисциплине "Проектирование высоконагруженных сервисов"

__Автор__ - [AlfaIV](https://github.com/alfaiv)</br>
__Задание__ - [Методические указания](https://github.com/init/highload/blob/main/homework_architecture.md)</br>
__[Таблица отчетности](https://vk.com/away.php?to=https%3A%2F%2Fdocs.google.com%2Fspreadsheets%2Fd%2F1XhRgeQARIXGoiTSYQStW4GTlXJ62-jgL36c2i8HvYJ4%2Fedit%3Fusp%3Dsharing&cc_key=)__

## Содержание

- [1. Функционал и продуктовые метрики](#chapter-one)
- [2. Расчет нагрузки](#chapter-two)
- [3. Глобальная балансировка нагрузки](#chapter-tree)
- [4. Локальная балансировка нагрузки](#chapter-four)
- [Используемые источники](#sorce)

 


## 1. Функционал и продуктовые метрики
<a id="chapter-one"></a>


__Функциональные возможности__ - это поисковая система. Она позволяет пользователям искать информацию в интернете, включая веб-страницы, изображения, видео, новости и другие типы контента.

__Целевая аудитория__ - в результате исследования данной темы, конкретного количества пользователей поисковый системы Яндекс не было найдено. Поэтому использовалась следующая логика. В основном аудитория данного сервиса - _русскоязычные пользователи интернета_ - рунет. [Согласно статистике](https://www.web-canape.ru/business/statistika-interneta-i-socsetej-na-2023-god-cifry-i-trendy-v-mire-i-v-rossii/) в России 127.6 миллионов пользователей Интернета. Согласно сервису [Яндекс Радар](https://radar.yandex.ru/search?period=all&selected_rows=iHMJ0E%252CvDPqTi%252C8fg2y0%252Ctmkupd%252CMCj1FA) пользователи Яндекса составляют 64.25% от общего числа пользователей поисковых систем. Таким образом, можно предположить, что количество пользователей составляет ``127.6E6*0.64 = 76E6``.</br>

Ниже представлена таблица, показывающая долю пользователей Яндекс из разных стран:

 Страна         |  Процент пользователей Яндекс поиска
----------------|------------------------
 Россия         |   95.04%
 Украина        |    0.89%
 Беларусь       |    0.66%
 Нидерланды     |    0.27%
 Молдова        |    0.27%
 Другие         |    2.87%

Для определения метрик DAU и MAU возьмем данные с сайта [HypeStat](https://hypestat.com/info/ya.ru):

- количество пользователей ~ 81,66M
- MAU ~ 44,9M
- DAU ~ 15,8M

__Ключевой функционал:__

- Регистрация и авторизация
- Поиск веб-страниц
- История поиска


## 2. Расчет нагрузки
<a id="chapter-two"></a>

__Продуктовые метрики__

Среднее количество действий пользователя можно оценить следующим образом. Т.к. данный сервис представляет собой поисковую систему, то вероятнее всего, что в общем случае сценарий использования сервиса следующий: пользователей заходит на главную поисковую страницу, набирает запрос и пользуется пагинатором. Согласно [5] страниц за визит 5,8 значит 3-4 перехода пользователь проводит за сеанс использования ресурса.</br>

Прикинем размер истории пользователя. Допустим, символы в истории пользователя кодируются UTF-8, тогда размер символа - 1 байт. 2 000 символов - максимальный размер URL адреса. _По собственному опыту_ ежедневного использования браузера, имею порядка 100 000 записей в истории. Таким образом, ``1 байт *2 000 символов * 100 000 записей = 200Гб``. Предположим, что история сжимается каким-либо алгоритмом, в среднем на 40%. Таким образом, получим следующий результат ``200Гб * (1 - 0.4) = 120Гб``

Таким образом, имеем следующие продуктовые метрики:

 Продуктовая метрика                      |Значение
------------------------------------------|--------
 MAU                                      | 44,9М  
 DAU                                      | 15,8М
 Средний размер страницы                  | 3 Мб
 Количество заходов на сайт за месяц      | 500M
 Записи истории пользователя              | 100 000 записей

Оценка объема страницы яндекс поиска.
![Размер стартовой страницы](img/yandex.png)
![Размер поисковой страницы](img/search.png)

Оценим размер Рунета
Согласно [19], рунет содержит ~ 5 миллионов доменов. Предположим, что средний размер ресурса 1.5 Мб, тогда размер рунета ~ 7 Тб.

__Технические метрики__

При загрузке страницы ya.ru инструменты разработчика показали, размер страницы в среднем 3 Мб. Учитывая перемещение пользователя по 5-6 страницам, получим трафик на одного пользователя. Умножив это значение на DAU, получим _суммарный суточный трафик_. Однако, учтем, что страницы у пользователя кешируются, поэтому значительным будет трафик получаемый уникальным пользователем (первая страница), далее они значительно снижается.</br>
_Пиковый трафик_ определим, как усредненный суточный с небольшим коэффициентом, например 1.5</br>
_RPS на авторизацию_ найдем из из следующих соображений соображений, поделим DAU на секунды в дне.<br>
При расчете сетевого трафика не будем учитывать запросы, связанные с _регистрацией пользователей_, так как они не создают ощутимую нагрузку на сервис. Основная нагрузка приходится на запросы к поисковой системе.


 Техническая метрика                             |Значение
-------------------------------------------------|------------------------
 Пиковое потребление трафика в течении суток     | 6.9 Гбит/с
 Суммарный суточный трафик                       | 49 702.5 Гбайт/сутки
 RPS (авторизация)                               | 183 запросов в секунду
 RPS                                             | 5 788 запросов в секунду
 Суммарный объем данных [11] *                   | 50 Пб
 Размер индекса [8]                              | 250 ТБ
 Размер истории пользователя                     | 120Гб

*Суммарный объем данных включает в себя и реплику и индексы.


## 3. Глобальная балансировка нагрузки
<a id="chapter-tree"></a>

Исходя из данных, представленных в первом пункте курсовой работы, можно сделать вывод, что нужно расположить большую часть датацентров в Росиии.<br>

Так как население России распределено неравномерно, нам потребуются расположить основную часть датацентров в европейской части страны[14].<br>

![плотность населения России](https://upload.wikimedia.org/wikipedia/commons/thumb/3/38/Federal_subjects_of_Russia_by_population_dencity.svg/1920px-Federal_subjects_of_Russia_by_population_dencity.svg.png)

Следовательно, были выбраны следующие города для установки датацентров:

- Москва
- Владимир
- Рязань


Также имеет смысл разместить сервера на Украине, например в Киеве (без учета текущей ситуации). Таким образом, можно улучшить качество предоставляемых услуг для жителей Украины, Молдовы, Белоруссии и южных регионов России.</br>

Таким образом, можно составить следующую карту датацентров: [карта датацентов](https://yandex.ru/maps/?um=constructor%3A1a245cfdfa0b8edad4e6334f166dc8c5658ea634ecf4b590220d70426b31da7d&source=constructorLink)


Для __глобальной балансировки запросов и нагрузки__ будем использовать:

- Применим latency-based DNS , так как он позволяет эффективно распределять траффик по RTT к ближайшим серверам


## 4. Локальная балансировка нагрузки
<a id="#chapter-four"></a>


__Kubernetes__
Kubernetes решает проблему отказоустойчивости в рамках сервисов. Он предоставляет механизмы для автоматического перезапуска контейнеров в случае их сбоя и обеспечивает механизмы для обнаружения и восстановления. Это позволяет быстро восстанавливать работоспособность сервисов и минимизировать простои.

__BGP/RIP балансировка__
Для распределения данных на балансировщик L3 будет использоваться BGP маршрутизация.

__L7 балансировщик__
Для балансировки запросов на сервисы, поднятые в подах Kubernetes, будет использоваться Nginx с помощью Weighted Least Connections.


__SSL termination__
Для снятия нагрузки с серверов по расшифровке SSL будет использоваться SSL Termination, который будет выполняться L7 балансировщиком.
Для кеширования сессии будет использоваться механизм Session cache.


## 5. Логическая схема БД
<a id="#chapter-five"></a>

__Структура базы данных__

Ниже представлены таблицы, поля и связи между ними без привязки к конкретным базам и шардингу.

![Структура БД](img/DBchart.drawio.svg)

Описание таблицы

 Таблица | Описание
---------|--------------------------------
 Users   | Хранение данных о пользователе
 History | История пользователя
 Indexes | Поисковые индексы
 Pages   | Проиндексированные страницы

__Размер данных__

__Нагрузска на запись/чтение__

Воспользуемся данными, полученными из 2 раздела.</br>
Для таблицы Users запросы на чтение примем как запросы на авторизацию, запросы на запись примем равными 5.</br>
Для таблиц History запросы на чтение примем рассчитаем следующим образом, предположим, что история пользователя синхронизируется (как пишеться, так и считывается) 1 раз в день. Таким образом, количество запросов равной количеству запросов на авторизацию.</br>
Для таблиц Index и Pages запросы на чтение примем как запросы на поиск (обращение к сервису), запросы на запись примем равными .</br>

 Таблица       | Чтение                     | Запись
---------------|----------------------------|----------------------------
 Users         | 183 запросов в секунду     | 5 запросов в секунду  
 History       | 183 запросов в секунду     | 183 запросов в секунду  
 Indexes       | 5 788 запросов в секунду   | запросов в секунду  
 Pages         | 5 788 запросов в секунду   | запросов в секунду  

## Используемые источники
<a id="sorce"></a>

1. [Статистика пользователей интернета](https://www.web-canape.ru/business/statistika-interneta-i-socsetej-na-2023-god-cifry-i-trendy-v-mire-i-v-rossii/)
1. [Яндекс инвесторам](https://ir.yandex.ru/)
1. [Яндекс Радар](https://radar.yandex.ru/search?period=all&selected_rows=iHMJ0E%252CvDPqTi%252C8fg2y0%252Ctmkupd%252CMCj1FA)
1. [Be1.ru](https://be1.ru/stat/ya.ru)
1. [SpyMetrics](https://spymetrics.ru/ru/website/ya.ru)
1. [HypeStat](https://hypestat.com/info/ya.ru)
1. [similarweb](https://www.similarweb.com/website/ya.ru/)
1. [Архитектура Поиска Яндекса](https://habr.com/ru/companies/yandex/articles/204282/)
1. [Что-то про БД](https://habr.com/ru/articles/123884/)
1. [Технологии Яндекса](https://yandex.ru/company/technologies/)
1. [Доклад про Яндекс](https://habr.com/ru/companies/yandex/articles/312716/)
1. [Что-то про поиск](https://habr.com/ru/companies/hh/articles/413261/)
1. [Статистика](https://datareportal.com/search?q=DIGITAL%202023%3A%20Rus)
1. [Контент Рунета](https://yandex.ru/company/researches/2009/ya_content_09/)
1. [Статистика по Яндексу](https://inclient.ru/yandex-stats/)
1. [Плотность населения России](https://ru.wikipedia.org/wiki/Плотность_населения_субъектов_Российской_Федерации)
1. [Инфраструктура Яндекса](https://www.tadviser.ru/index.php/Статья:Яндекс_(ИТ-инфраструктура))
1. [Конструктор Карт](https://yandex.ru/map-constructor/)
1. [Количество доменов РУ](https://www.kommersant.ru/doc/5773611?tg)

#### Дополнительно

https://habr.com/ru/companies/vk/articles/149498/
https://xakep.ru/2014/01/11/big-data-secrets/
https://habr.com/ru/companies/hh/articles/413261/
[Яндекс MapReduce](https://habr.com/ru/companies/yandex/articles/311104/)
[Про mail поиск](https://habr.com/ru/companies/vk/articles/167297/)
