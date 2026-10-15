# goit-rdb-hw-07 — Time-series features, JSONB event-log, GIN vs B-tree

Домашнє завдання 7. Notebook виконує такі кроки:

- рахує time-series ML-ознаки для pickup-зон NYC Taxi через window functions: lag, rolling і expanding;
- створює синтетичний JSONB event-log і запити до нього через `@>`, `?`, `?|` та JSONPath;
- вимірює ефект GIN-індексу і expression B-tree через `EXPLAIN (ANALYZE, BUFFERS)`;
- порівнює JSONB-підхід із нормалізованою таблицею.

## Джерела даних

**NYC TLC Yellow Taxi Trip Records, лютий і березень 2024.**

- Офіційна сторінка: https://www.nyc.gov/site/tlc/about/tlc-trip-record-data.page
- Файли:
  - https://d37ci6vzurychx.cloudfront.net/trip-data/yellow_tripdata_2024-02.parquet
  - https://d37ci6vzurychx.cloudfront.net/trip-data/yellow_tripdata_2024-03.parquet
- Колонки: `tpep_pickup_datetime`, `tpep_dropoff_datetime`, `PULocationID`, `DOLocationID`, `trip_distance`, `fare_amount`, `passenger_count`.

**Синтетичний event-log** `t7_hw_events`: 50 000 подій п'яти типів, згенерованих детерміновано через `generate_series`, без `RANDOM()`.

### Відхилення від шаблону

- **Два місяці замість `head(500_000)` одного.** 500 000 перших рядків березня покривають лише 4–5 днів. На такій історії `LAG(..., 30)` завжди `NULL`, і view з фільтром `trips_lag_30d IS NOT NULL` порожній. Лютий і березень разом дають 60 днів, як у stub-схемі.
- **Фільтр періоду 2024-02-01 … 2024-03-31.** У TLC-файлах трапляються поїздки з «чужими» датами (2002, 2009 тощо), їх відкидаємо.
- **Densification `t7_hw_taxi_daily`.** Відсутні пари (зона, день) додаються з `trip_count = 0`. Без цього `LAG(n)` рахував би рядки, а не дні, і для малих зон «lag-7d» брав би значення з іншої дати.
- **Завантаження через COPY.** `to_sql(..., method=psql_insert_copy)` пише дані через `COPY FROM STDIN`, бо після очищення в двох місяцях ≈ 5–6 млн рядків.
- **`DROP TABLE ... CASCADE` для `t7_hw_taxi_daily`.** `pgserver` зберігає базу між перезапусками kernel, і без `CASCADE` повторний Run All падає на залежному view.

### Stub-schema fallback

Якщо TLC недоступний, notebook автоматично використовує stub-схему з умови: 60 днів × 5 зон = 300 рядків, фіксована дата `2024-03-31`. Примусово її можна ввімкнути змінною `HW7_FORCE_STUB=1`. Обраний режим виводиться як `DATA_MODE`.

## Структура

```
goit-rdb-hw-07/
├── README.md
├── hw7_timeseries_jsonb.ipynb
├── .gitignore
└── data/            # кеш Parquet-файлів (не комітиться)
```

## Запуск

Потрібен **Python 3.10–3.12**, бо `pgserver` не має збірок для Python 3.13+.

```bash
git clone <repo-url> goit-rdb-hw-07
cd goit-rdb-hw-07
python3.12 -m venv .venv && source .venv/bin/activate   # macOS: brew install python@3.12
pip install jupyter
jupyter notebook hw7_timeseries_jsonb.ipynb
```

Далі виконайте **Kernel → Restart & Run All**. Перша комірка сама встановлює залежності. Parquet-файли (≈ 50 МБ кожен) завантажуються в `data/` один раз, при наступних запусках береться кеш.

**Colab:** runtime має бути з Python ≤ 3.12 (Runtime → Change runtime type → Runtime version).

## Порядок виконання

1. Запуск `pgserver` (`/tmp/hw7_pg`), завантаження TLC або stub-схеми, побудова `t7_hw_taxi_daily` і densification.
2. Ознаки lag-1d/7d/30d, rolling 7d avg fare, rolling 30d std fare, expanding sum/avg. Окремо створюється view `v_t7_hw_taxi_features` і перевіряється `PARTITION BY` у всіх вікнах. Є демонстрація помилки без `PARTITION BY`.
3. `t7_hw_events` (JSONB), containment, key existence, JSONPath.
4. `EXPLAIN` до і після GIN (`jsonb_path_ops`, додатково `jsonb_ops` для `?`), до і після expression B-tree, вартість запису з різними індексами, зведена таблиця планів.
5. `t7_hw_checkouts_norm`, порівняння JSONB і normalized через `EXPLAIN`, MongoDB-pseudocode, reflection.
