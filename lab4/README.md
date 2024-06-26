# Применение технологий ИИ и машинного обучения для поиска угроз ИБ
Zhidkov Georgy

Лабораторная работа №4

> Анализ данных сетевой активности с использованием аналитической
> in-memory СУБД DuckDB

## Цель

1.  Изучить возможности СУБД DuckDB для обработки и анализ больших
    данных
2.  Получить навыки применения Arrow совместно с языком программирования
    R
3.  Получить навыки анализа метаинфомации о сетевом трафике
4.  Получить навыки применения облачных технологий хранения, подготовки
    и анализа данных: Yandex Object Storage, Rstudio Server.

## Исходные данные

1.  ОС Windows 11
2.  RStudio Server
3.  Yandex Cloud: S3 Object Storage
4.  СУБД DuckDB
5.  Датасет tm_data.pqt

## Описание работы

### Общая ситуация

Вы – специалист по информационной безопасности компании “СуперМегатек”.
Вы, являясь специалистом Threat Hunting, часто используете информацию о
сетевом трафике для обнаружения подозрительной и вредоносной активности.
Помогите защитить Вашу компанию от международной хакерской группировки
AnonMasons.

У Вас есть данные сетевой активности в корпоративной сети компании
“СуперМегатек”. Данные хранятся в Yandex Object Storage.

### Задание

Используя язык программирования R, СУБД и пакет duckdb и облачную IDE
Rstudio Server, развернутую в Yandex Cloud, выполнить задания и
составить отчёт.

## Ход работы

### Подключение к RStudio Server

Аналогично предыдущей практической подключаемся к RStudio Server:
`ssh -i <путь-к-ключу> -L 8787:127.0.0.1:8787 user14@62.84.123.211`

### Установка библиотек

``` r
library(duckdb)
```

    Loading required package: DBI

``` r
library(dplyr)
```


    Attaching package: 'dplyr'

    The following objects are masked from 'package:stats':

        filter, lag

    The following objects are masked from 'package:base':

        intersect, setdiff, setequal, union

``` r
library(tidyverse)
```

    ── Attaching core tidyverse packages ──────────────────────── tidyverse 2.0.0 ──
    ✔ forcats   1.0.0     ✔ readr     2.1.5
    ✔ ggplot2   3.4.4     ✔ stringr   1.5.1
    ✔ lubridate 1.9.3     ✔ tibble    3.2.1
    ✔ purrr     1.0.2     ✔ tidyr     1.3.1
    ── Conflicts ────────────────────────────────────────── tidyverse_conflicts() ──
    ✖ dplyr::filter() masks stats::filter()
    ✖ dplyr::lag()    masks stats::lag()
    ℹ Use the conflicted package (<http://conflicted.r-lib.org/>) to force all conflicts to become errors

### Настройка СУБД и импорт датасета

``` r
con <- dbConnect(duckdb::duckdb(), dbdir = ":memory:")
dbExecute(conn = con, "INSTALL httpfs; LOAD httpfs;")
```

    [1] 0

``` r
parquet_file = "https://storage.yandexcloud.net/arrow-datasets/tm_data.pqt"

query <- "SELECT * FROM read_parquet([?])"
df <- dbGetQuery(con, query, list(parquet_file))

df %>% head
```

         timestamp           src          dst port bytes
    1 1.578326e+12   13.43.52.51 18.70.112.62   40 57354
    2 1.578326e+12 16.79.101.100  12.48.65.39   92 11895
    3 1.578326e+12 18.43.118.103  14.51.30.86   27   898
    4 1.578326e+12 15.71.108.118 14.50.119.33   57  7496
    5 1.578326e+12  14.33.30.103  15.24.31.23  115 20979
    6 1.578326e+12 18.121.115.31  13.56.39.74   92  8620

### Выполнение заданий

#### Задание 1. Найдите утечку данных из Вашей сети

Важнейшие документы с результатами нашей исследовательской деятельности
в области создания вакцин скачиваются в виде больших заархивированных
дампов. Один из хостов в нашей сети используется для пересылки этой
информации – он пересылает гораздо больше информации на внешние ресурсы
в Интернете, чем остальные компьютеры нашей сети. Определите его
IP-адрес.

``` r
df_1 <- df %>%
  filter(!grepl('^1[2-4].*', dst)) %>%
  group_by(src) %>%
  summarise(sum_bytes = sum(bytes)) %>%
  top_n(n = 1, wt = sum_bytes)

df_1 %>% collect()
```

    # A tibble: 1 × 2
      src            sum_bytes
      <chr>              <dbl>
    1 13.37.84.125 10625497574

Ответ: 13.37.84.125

#### Задание 2. Найдите утечку данных 2

Другой атакующий установил автоматическую задачу в системном
планировщике cron для экспорта содержимого внутренней wiki системы. Эта
система генерирует большое количество трафика в нерабочие часы, больше
чем остальные хосты. Определите IP этой системы. Известно, что ее IP
адрес отличается от нарушителя из предыдущей задачи.

Определим рабочие и нерабочие часы:

``` r
work_time <- df %>%
  filter(!grepl('^1[2-4].*', dst)) %>%
  mutate(timestamp = hour(as_datetime(timestamp / 1000))) %>% 
  group_by(timestamp) %>% 
  summarize(sum_bytes = sum(bytes)) %>% 
  arrange(desc(sum_bytes))

work_time %>% collect()
```

    # A tibble: 24 × 2
       timestamp   sum_bytes
           <int>       <dbl>
     1        23 82031837646
     2        16 81994628850
     3        18 81992814764
     4        21 81963711653
     5        22 81888013879
     6        19 81861371178
     7        20 81848190437
     8        17 81841962148
     9         7  3300183251
    10        12  3100273165
    # ℹ 14 more rows

``` r
df_2 <- df %>%
  mutate(timestamp = hour(as_datetime(timestamp / 1000))) %>%
  filter(!grepl('^1[2-4].*', dst) & timestamp >= 0 & timestamp <= 15) %>%
  group_by(src) %>%
  summarise(sum_bytes = sum(bytes)) %>%
  filter(src != "13.37.84.125") %>%
  top_n(1, wt = sum_bytes)

df_2 %>% collect()
```

    # A tibble: 1 × 2
      src         sum_bytes
      <chr>           <int>
    1 12.55.77.96 289566918

Ответ: 12.55.77.96

#### Задание 3. Найдите утечку данных 3

Еще один нарушитель собирает содержимое электронной почты и отправляет в
Интернет используя порт, который обычно используется для другого типа
трафика. Атакующий пересылает большое количество информации используя
этот порт, которое нехарактерно для других хостов, использующих этот
номер порта. Определите IP этой системы. Известно, что ее IP адрес
отличается от нарушителей из предыдущих задач.

``` r
df_3 <- df %>%
  filter(!grepl('^1[2-4].*', dst) & src != "13.37.84.125" & src != "12.55.77.96") %>%
  group_by(src, port) %>%
  summarise(bytes_ip_port = sum(bytes), .groups = "drop") %>%
  group_by(port) %>%
  mutate(sum_traffic_by_port = mean(bytes_ip_port)) %>%
  ungroup() %>%
  top_n(1, bytes_ip_port / sum_traffic_by_port)

df_3 %>% collect()
```

    # A tibble: 1 × 4
      src          port bytes_ip_port sum_traffic_by_port
      <chr>       <int>         <int>               <dbl>
    1 12.30.96.87   124        356207              20601.

Ответ: 12.30.96.87

#### Задание 4. Обнаружение канала управления

Зачастую в корпоративных сетях находятся ранее зараженные системы,
компрометация которых осталась незамеченной. Такие системы генерируют
небольшое количество трафика для связи с панелью управления бот-сети, но
с одинаковыми параметрами – в данном случае с одинаковым номером порта.
Какой номер порта используется бот-панелью для управления ботами?

``` r
df_4 <- df%>%
  group_by(port) %>%
  summarise(min_bytes = min(bytes),
            max_bytes = max(bytes),
            diff_bytes = max(bytes) - min(bytes),
            avg_bytes = mean(bytes),
            count = n()) %>%
  filter(avg_bytes - min_bytes < 10 & min_bytes != max_bytes) %>%
  select(port)

df_4 %>% collect()
```

    # A tibble: 1 × 1
       port
      <int>
    1   124

Ответ: 124

#### Задание 5. Обнаружение P2P трафика

Иногда компрометация сети проявляется в нехарактерном трафике между
хостами в локальной сети, который свидетельствует о горизонтальном
перемещении (lateral movement). В нашей сети замечена система, которая
ретранслирует по локальной сети полученные от панели управления бот-сети
команды, создав таким образом внутреннюю пиринговую сеть. Какой
уникальный порт используется этой бот сетью для внутреннего общения
между собой?

``` r
df_5 <- df %>%
  filter(grepl('^1[2-4].*', src) & grepl('^1[2-4].*', dst)) %>%
  group_by(port) %>%
  summarise(diff_bytes = max(bytes) - min(bytes)) %>%
  arrange(desc(diff_bytes)) %>% 
  select(port) %>%
  head(1)

df_5 %>% collect()
```

    # A tibble: 1 × 1
       port
      <int>
    1   115

Ответ: 115

#### Задание 6. Чемпион малвари

Нашу сеть только что внесли в списки спам-ферм. Один из хостов сети
получает множество команд от панели C&C, ретранслируя их внутри сети. В
обычных условиях причин для такого активного взаимодействия внутри сети
у данного хоста нет. Определите IP такого хоста.

``` r
df_6 <- df %>%
  filter(grepl('^1[2-4].*', src) & grepl('^1[2-4].*', dst)) %>%
  group_by(src) %>%
  summarise(count = n()) %>%
  arrange(desc(count)) %>%
  head(1)

df_6 %>% collect()
```

    # A tibble: 1 × 2
      src         count
      <chr>       <int>
    1 13.42.70.40 65109

Ответ: 13.42.70.40

``` r
dbDisconnect(con, shutdown=TRUE)
```

## Оценка результата

В результате практической работы был проведен анализ сетевой активности
с помощью Apache Arrow и DuckDB и были найдены проблемы во внутренней
сети предприятия

## Вывод

Были получены навыки использования СУБД DuckDB для обработки и анализа
больших данных совместно с языком программирования R
