# goit-rdb-hw-04 — DML та DDL команди. Складні SQL-вирази

Домашнє завдання 4 (Тема 4) курсу реляційних БД: багатотабличний e-commerce-датасет Olist у PostgreSQL —
raw- і typed-шари зі constraints, DML з `RETURNING` / `ON CONFLICT` / `UPDATE ... FROM`, customer-level сегментація,
JOIN-запити, агрегування, set operations і window functions.

## Джерело даних

**Olist Brazilian E-commerce Public Dataset** — Kaggle:
[olistbr/brazilian-ecommerce](https://www.kaggle.com/datasets/olistbr/brazilian-ecommerce)
(≈100 тис. замовлень бразильського маркетплейсу Olist, 2016–2018, ліцензія CC BY-NC-SA 4.0).

## Використані таблиці

Використано **повний датасет без семплювання**: 6 таблиць із 9 (геолокація, платежі та переклад категорій не потрібні для завдання).

| CSV | Raw-таблиця | Typed-таблиця | Ключ | Рядків |
|---|---|---|---|---:|
| `olist_customers_dataset.csv` | `olist_customers_raw` | `olist_customers` | `customer_id` | 99 441 |
| `olist_orders_dataset.csv` | `olist_orders_raw` | `olist_orders` | `order_id` | 99 441 |
| `olist_order_items_dataset.csv` | `olist_order_items_raw` | `olist_order_items` | `(order_id, order_item_id)` | 112 650 |
| `olist_products_dataset.csv` | `olist_products_raw` | `olist_products` | `product_id` | 32 951 |
| `olist_sellers_dataset.csv` | `olist_sellers_raw` | `olist_sellers` | `seller_id` | 3 095 |
| `olist_order_reviews_dataset.csv` | `olist_order_reviews_raw` | `olist_order_reviews` | технічний `review_row_id` (identity); `review_id` — джерельний атрибут | 99 224 |

Похідні таблиці: `olist_customer_dim` (PK `customer_unique_id`), `customer_segments`, `seller_score`, `seller_alerts`.

## Отримання файлів

Notebook знаходить CSV автоматично, без ручних дій:

1. **З репозиторію** — файли лежать у [`data/olist/`](data/olist) (≈56 МБ, кожен файл < 100 МБ, тому вкладаються в ліміт GitHub).
   Використовуються, якщо notebook запущено з кореня репозиторію або репозиторій склоновано в Colab у `/content/goit-rdb-hw-04`.
2. **Через KaggleHub** — інакше виконується `kagglehub.dataset_download('olistbr/brazilian-ecommerce')`.
   Датасет публічний, тож Kaggle credentials не потрібні.

## Структура репозиторію

```
.
├── hw4_olist_dml_ddl_joins.ipynb   # notebook з output усіх клітинок
├── README.md
└── data/
    └── olist/                      # 6 CSV Olist (копія з Kaggle)
```

## Вміст notebook

1. **Setup і typed-схема** — запуск PostgreSQL, версія сервера, raw-шар через pandas `to_sql`, typed-таблиці з PK / FK / CHECK / NOT NULL, `INSERT ... SELECT`, smoke-test кількості рядків.
2. **DML** — `INSERT ... RETURNING` (`seller_alerts`), `INSERT ... ON CONFLICT DO UPDATE` (`seller_score` на грануляності `seller_id × order_id`), `UPDATE ... FROM` (`is_late`), `DELETE ... RETURNING`; перевірка ідемпотентності.
3. **`customer_segments`** — `olist_customer_dim`, DDL з identity PK, FK, UNIQUE, CHECK, NOT NULL, DEFAULT; RFM-сегментація на рівні `customer_unique_id` через `INSERT ... SELECT ... ON CONFLICT`.
4. **JOIN-запити** — INNER (4 таблиці), LEFT + COALESCE, FULL OUTER, SELF, anti-join (`LEFT JOIN ... IS NULL` і `NOT EXISTS`) у форматі business-question / SQL / interpretation.
5. **GROUP BY / HAVING** — revenue за категоріями, AOV за штатами, `STRING_AGG` / `ARRAY_AGG` для повторних покупців, `WHERE` vs `HAVING`.
6. **Set operations і window functions** — `UNION`, `INTERSECT`, `EXCEPT`, `ROW_NUMBER()`, `RANK()`; data-quality огляд; Reflection.

## Запуск

### Google Colab

Відкрити `hw4_olist_dml_ddl_joins.ipynb` у Colab → `Runtime → Restart session and run all`.

- Перша клітинка пробує `pgserver`; якщо для поточного Python у Colab немає його збірки, автоматично піднімає системний PostgreSQL
  через `apt-get install postgresql`. Далі notebook працює лише з `engine`.
- Дані завантажуються через KaggleHub. Щоб використати CSV з репозиторію, перед запуском виконати:
  `!git clone https://github.com/sofiiadorosh/goit-rdb-hw-04.git /content/goit-rdb-hw-04`.

### Локально

Потрібен Python 3.10–3.12 (для `pgserver`):

```bash
python3.12 -m venv .venv && source .venv/bin/activate
pip install pgserver psycopg2-binary sqlalchemy pandas kagglehub jupyter
jupyter nbconvert --to notebook --execute --inplace hw4_olist_dml_ddl_joins.ipynb
```

Якщо PostgreSQL уже запущено, можна не встановлювати `pgserver`, а передати адресу сервера через змінну середовища:

```bash
HW4_DB_URI=postgresql://postgres@127.0.0.1:5432/postgres jupyter nbconvert --to notebook --execute --inplace hw4_olist_dml_ddl_joins.ipynb
```

Notebook ідемпотентний: усі таблиці перестворюються через `DROP TABLE IF EXISTS ... CASCADE`, а службові таблиці наповнюються через `ON CONFLICT`,
тому повторний *Run All* дає той самий результат.
