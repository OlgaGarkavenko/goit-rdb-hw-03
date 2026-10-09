# goit-rdb-hw-03 — Seoul Bike Sharing Demand у PostgreSQL

Домашнє завдання 3: повний цикл роботи з сирим CSV — завантаження в PostgreSQL через `COPY FROM STDIN`, staging- і clean-рівні, аудит якості даних, аналіз розподілів та аномалій, 11 DQL/EDA-запитів і reflection.

## Датасет

| | |
|---|---|
| Назва | **Seoul Bike Sharing Demand** |
| Офіційне джерело | UCI Machine Learning Repository, id = 560: <https://archive.ics.uci.edu/dataset/560/seoul+bike+sharing+demand> |
| Файл, що завантажується | `https://archive.ics.uci.edu/static/public/560/seoul+bike+sharing+demand.zip` → `SeoulBikeData.csv` |
| Період | 01.12.2017 – 30.11.2018, погодинно |
| Кількість рядків | **8 760** (365 днів × 24 години), 14 колонок |
| Ліцензія | CC BY 4.0 |

## Спосіб формування вибірки

Вибірка не робиться - датасет малий, тому використовуються **всі 8 760 рядків**.

1. Notebook завантажує офіційний ZIP з UCI в `data/raw/` (лише якщо його там ще немає).
2. `SeoulBikeData.csv` декодується явно (UTF-8, а при невдачі — cp1252, бо заголовки джерела містять символ `°`) і читається з `dtype=str`, щоб значення не змінювалися.
3. Колонки позиційно перейменовуються в `snake_case` з суфіксом `_raw`, результат зберігається як UTF-8 CSV: `data/hw3_seoul_bike.csv`.
4. CSV завантажується через `COPY ... FROM STDIN` у `hw3_bike_staging` (усі колонки `TEXT`).
5. `INSERT INTO hw3_bike_rentals ... SELECT` очищує та приводить типи (`DATE`, `TIMESTAMPTZ` у поясі `Asia/Seoul`, `NUMERIC(p,s)`, `BOOLEAN`, `CHECK`, `UNIQUE (rental_date, rental_hour)`).

## Структура репозиторію

```
goit-rdb-hw-03/
├── hw3_seoul_bike.ipynb     # notebook з output усіх клітинок
├── data/
│   └── hw3_seoul_bike.csv   # контрольований CSV (8 760 рядків, ~0,6 МБ)
└── README.md
```

## Запуск

1. Відкрити `hw3_seoul_bike.ipynb` у Google Colab (*File → Upload notebook* або з GitHub).
2. *Runtime → Restart session and run all*.

Notebook сам встановлює залежності (`pgserver`, `psycopg2-binary`, `sqlalchemy`, `pandas`, `pyarrow`, `ucimlrepo`), піднімає PostgreSQL через `pgserver` у `/tmp/hw3_pg`, завантажує дані з UCI і виконує всі кроки. Ручне редагування шляхів чи SQL не потрібне.

> **Примітка щодо драйвера.** SQLAlchemy 2.1+ для URI `postgresql://` за замовчуванням використовує `psycopg` (v3). Щоб працювати з `psycopg2-binary`, notebook підставляє `postgresql+psycopg2://` в URI від `pgserver`.

## Таблиці

| Таблиця | Призначення |
|---|---|
| `hw3_bike_staging` | сирий імпорт, 14 колонок `TEXT` |
| `hw3_bike_rentals` | типізована clean-таблиця з identity-PK, `NOT NULL`, `CHECK` і `UNIQUE` |
| `hw3_to_num(text)` | допоміжна SQL-функція: безпечне приведення тексту до `NUMERIC` (нечислове → `NULL`) |
