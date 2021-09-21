# Изучение обзервера и выложенного дампа ДЭГ по Москве

Судя по обзерверу [1], расшифровка голосов в блокчейне для одномандатных округов была завершена (или остановлена) около 21:19.

Выложенный дамп [2] от 20 сентября (06:00) содержит 2021969 выданных бюллетеней и столько же добавленных ключей шифрования; 1987373 полученных бюллетеней; но лишь 1319943 расшифрованных бюллетеней.

Результаты среди расшифрованных бюллетеней: [results_with_names.csv](results_with_names.csv).

Также один бюллетень испорчен.

# Повторение результатов

- Скачать дамп [2], убедиться в правильности хэш-суммы:
``` sh
sha256sum observer-20210920_060000.sql.gz
```

- Подготовить БД (предполагается, что Postresql 13 установлен и запущен), выполнив в `sudo -u postgres psql`:
``` sql
CREATE USER deg;
CREATE DATABASE mos1 OWNER deg;
```

- Выложить дамп в БД (потребуется около 20 Гб места на диске и полчаса времени):
``` sh
zcat observer-20210920_060000.sql.gz | psql -U deg -d mos1
```

- Убедиться, что нет бюллетеней с двумя выбранными результатами (расшифрованный результат - массив, поэтому могло быть и такое):
``` sql
SELECT * FROM decrypted_ballots WHERE ARRAY_UPPER(decrypted_choice, 1) > 1;
```
(выдаёт 0 строк)

- Подсчитать количество расшифрованных бюллетеней:
``` sql
SELECT COUNT(*) FROM decrypted_ballots;
```
(выдаёт 1318977)

- Подсчитать количество транзакций по методам (запуск голосования, ввод id голосующих, выдача бюллетеня и т.п.):
``` sql
SELECT method_id, COUNT(*) FROM transactions GROUP BY method_id;
```
Результат:
```
 method_id |  count
-----------+---------
         0 |       1
         1 |   20148
         2 |       2
         4 | 1987373
         5 | 2021969
         6 | 2021969
         7 |       4
         8 |       1
         9 | 1319943
```

- Подсчитать результаты по кандидатам:
``` sql
SELECT decrypted_choice[1], COUNT(*) FROM decrypted_ballots GROUP BY decrypted_choice[1] ;
```

- Эти результаты нужно сохранить в csv-файл results.csv со столбцами decrypted_choice и count

- Получить соответствие id кандидатам:
``` sql
SELECT method_id, payload FROM transactions WHERE method_id=0;
```

- Сохранить ballots_config из payload в файл `ballots_config.json`

- Преобразовать соответствие id-кандидат в csv-файл `ballots_config.csv`:
``` sh
jq -r '[.[] | .district_id as $d | .options | to_entries | map({district: $d, key: .key, value: .value})] | add  | .[] | [.district, .key, .value] | @csv' ballots_config.json > ballots_config.csv
```

- Объединить результаты и кандидатов:
``` sh
mlr --csv join -f ballots_config.csv -j choice -r decrypted_choice then sort -f district,name results.csv > results_with_names.csv
```

## Ссылки

- [1] https://observer.mos.ru/all/servers/1/blocks
- [2] https://observer.mos.ru/1/dump/observer-20210920_060000.sql.gz, sha256: 495d1cc9136e26288208478dec98c702db73a40d93e8be76bb24d7d2be355883
- [3] jq: https://stedolan.github.io/jq/
- [4] mlr: https://miller.readthedocs.io/en/latest/
