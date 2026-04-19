# Data Extraction and Loading Pipeline

This project consists of a data pipeline using **Meltano** for extraction, **Apache Airflow** for orchestration, and **PostgreSQL** as the database, ensuring automation, traceability, and data reprocessing.

---

## Project Description

The challenge is to build a pipeline that:

- Extracts data from two sources: a PostgreSQL database (Northwind) and a CSV file.
- Writes data locally, organized by source (csv or postgres), table, and execution date.
- Loads data from local storage into a PostgreSQL database.
- Ensures data is processed independently and traceably, with support for reprocessing previous dates.

---

## 🛠️ Tools Used

| Tool | Version |
|------|---------|
| Python | 3.11.5 |
| Meltano | 3.6.0 |
| Apache Airflow | 2.10.4 |
| PostgreSQL | Northwind DB + northwind_processed (destination) |

---

## Environment Setup

### 1 - Install Dependencies

Clone the repository and install the Python dependencies:

```bash
git clone https://github.com/TinyHero13/code-challenge-indicium.git
cd code-challenge-indicium
pip install -r requirements.txt
```

Navigate to the Meltano directory and install its dependencies:

```bash
cd meltano_elt
meltano install
```

---

### 2 - Configure Apache Airflow

Initialize Airflow with the commands below:

```bash
meltano invoke airflow:initialize
meltano invoke airflow users create -u admin@localhost -p password --role Admin -e admin@localhost -f admin -l admin
meltano invoke airflow scheduler
meltano invoke airflow webserver
```

> The Airflow interface will be available at: `http://localhost:8080`

---

### 3 - Configure the PostgreSQL Database

Start the Northwind database from `docker-compose.yml`:

```bash
docker-compose up -d
```

Create the destination database for processed data:

```sql
CREATE DATABASE northwind_processed;
```

Configure the Meltano `tap-postgres` (already included when installing Meltano dependencies in step 1):

```bash
meltano config tap-postgres set database northwind
meltano config tap-postgres set host localhost
meltano config tap-postgres set port 5432
meltano config tap-postgres set user northwind_user
meltano config tap-postgres set password thewindisblowing
```

Once configured, test it to verify everything is working:

```bash
meltano config tap-postgres test
```

![alt text](imgs/image1.png)

Also configure the `target-postgres`:

```bash
meltano config target-postgres set database northwind_processed
meltano config target-postgres set host localhost
meltano config target-postgres set port 5432
meltano config target-postgres set user northwind_user
meltano config target-postgres set password thewindisblowing
```

---

### 4 - Update the DAG Root Directory

For the DAG to work correctly, you need to adjust the root directory path of the project. The only required change is to update the `PROJECT_ROOT` variable in the `indicium_elt` DAG file to reflect the current path of the project in your environment:

```python
PROJECT_ROOT = '/your/current/project/directory'
```

---

### 5 - Run the DAG

After configuration, you can trigger the DAG either through the Airflow UI or via the command line:

```bash
meltano invoke airflow dags trigger indicium-northwind-elt
```

You can also trigger it for a previous date:

```bash
meltano invoke airflow dags trigger -e 2025-01-20 indicium-northwind-elt
```

![alt text](imgs/image2.png)

---

## Final Result

After the pipeline runs, data is loaded into the PostgreSQL database and organized into relational tables, enabling queries that combine tables that were not originally present in the source database.

For example, you can run a query that joins `order_details` with other tables:

![alt text](imgs/image3.png)

```sql
SELECT 
    od.order_id,
    o.ship_region,
    p.unit_price,
    p.quantity_per_unit,
    c.category_name,
    c.description
FROM 
    order_details od
LEFT JOIN 
    orders o ON o.order_id = od.order_id
LEFT JOIN 
    products p ON p.product_id = od.product_id
LEFT JOIN 
    categories c ON c.category_id = p.category_id;
```

Date-specific folders are also created for each execution:

![alt text](imgs/image4.png)
