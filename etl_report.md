
# 📝 Отчет по итоговому заданию ETL (4 модуль)

## 📦 Общая архитектура

- **Источник данных:** `transactions_v2.csv` (размещён в Object Storage Yandex Cloud)
- **ETL-пайплайн:** Airflow → Dataproc (PySpark) → Kafka → PostgreSQL (Managed Service)
- **Форматы:** CSV → Parquet → JSON → Kafka message → PostgreSQL

---

## 🔧 Компоненты пайплайна

### Задание 1. 💨 Работа с Yandex DataTransfer

- Вручную создана таблица в YDB
  <details>
    <summary>Тут SQL скрипт</summary>
  
    ### sql-скрипт создания таблицы в YDB
    ```sql
    CREATE TABLE transactions_v2 (
          msno Utf8,
          payment_method_id Int32,
          payment_plan_days Int32,
          plan_list_price Int32,
          actual_amount_paid Int32,
          is_auto_renew Int8,
          transaction_date Utf8,
          membership_expire_date Utf8,
          is_cancel Int8,
          PRIMARY KEY (msno)
      );
    ```
  </details> 
- В созданную таблицу с помощью CLI загружен датасет transaction_v2
  <details>
    <summary>Тут bash скрипт</summary>
  
    ### bash-скрипт загрузки датасета
    ```bash
    ydb  `
    --endpoint grpcs://ydb.serverless.yandexcloud.net:2135 `
    --database /ru-central1/b1g9tm1cvjc9r6hl0g83/etn4ciikjn2811hfpjo9 `
    --sa-key-file authorized_key.json `
    import file csv `
    --path transactions_v2 `
    --delimiter "," `
    --skip-rows 1 `
    --null-value "" `
    --verbose `
    transactions_v2.csv
    ```
  </details> 
- Создан трансфер данных с источником в YDB и приемником в Object Storage
  `s3a://etl-data-transform/transactions_v2.parquet`
    	<details>
    	<summary>Тут скриншоты</summary>
		- ![Скриншот](screenshots/screenshot1.jpg)
  		- ![Скриншот](screenshots/screenshot2.jpg)
  		- ![Скриншот](screenshots/screenshot3.jpg)
	</details> 

### Задание 2. 💨 Автоматизация работы с Yandex Data Processing при помощи Apache AirFlow

- Подготовлена инфраструктура (Managed service for Airflow)
- Создан DAG **DATA_INGEST**, который:
    - Создает Data Proc кластер.      
    - Запускает на кластере PySpark-задание для обработки Parquet-файла.
    - После завершения работы задания удаляет кластер.
- Очистка данных:
  - Приведение типов всех полей (`Integer`, `Boolean`, `Date`, `String`)
  - Удаление строк с пустыми значениями
  - Преобразование `transaction_date` и `membership_expire_date` из `yyyyMMdd` в `DateType`
- Результат сохраняется в формате Parquet:
  - `s3a://etl-data-transform/transactions_v2_clean.parquet`

### 2. 📤 Отправка в Kafka

- Скрипт `parquet-to-kafka-loop.py`:
  - Загружает Parquet-файл.
  - Каждую секунду выбирает 100 случайных строк.
  - Преобразует их в JSON.
  - Отправляет в Kafka-топик `dataproc-kafka-topic`.
- Kafka использует:
  - Протокол: `SASL_SSL`
  - Механизм: `SCRAM-SHA-512`

### 3. 📥 Чтение из Kafka и запись в PostgreSQL

- Стриминговое приложение на PySpark:
  - Читает сообщения из Kafka.
  - Десериализует JSON.
  - Преобразует даты.
  - Пишет в таблицу PostgreSQL `transactions_stream` с помощью `.foreachBatch()`.

---

## 📊 Визуализация в Yandex DataLens

**Настройки фильтрации:**
- 🔍 Фильтр по полю `msno`
- 📅 Сортировка по дате транзакции (сначала новые)

### 🔹 Дашборды и чарты

| Название чартa             | Источник        | Автор          | Дата     | Описание                                          |
|----------------------------|------------------|----------------|----------|---------------------------------------------------|
| 💳 Методы оплаты по сумме  | kafka-stream     | skyranger2008  | 15.06.25 | Топ-10 payment_method_id по `actual_amount_paid` |
| 📆 Сумма по месяцам         | kafka-stream     | skyranger2008  | 15.06.25 | Группировка по `transaction_date`, сумма         |
| 📆 Количество по месяцам    | kafka-stream     | skyranger2008  | 15.06.25 | Кол-во транзакций по месяцам                      |
| 🧾 Методы оплаты            | kafka-stream     | skyranger2008  | 15.06.25 | Распределение по `payment_method_id`             |

---

## ✅ Результаты выполнения

- ✅ Файл CSV успешно загружен, очищен, сохранён в Parquet
- ✅ Kafka-топик получает данные в режиме реального времени
- ✅ PostgreSQL таблица наполняется через стриминг из Kafka
- ✅ Данные визуализированы в Yandex DataLens
