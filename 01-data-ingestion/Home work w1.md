# 🚀 Module 1 Homework: Docker & SQL

A comprehensive guide to Docker, SQL, and Terraform concepts with practical examples and solutions.

---

## 📋 Table of Contents

- [Question 1: Understanding Docker Images](#question-1-understanding-docker-images)
- [Question 2: Docker Networking & Compose](#question-2-docker-networking--compose)
- [Data Preparation](#data-preparation)
- [Question 3: Counting Short Trips](#question-3-counting-short-trips)
- [Question 4: Longest Trip](#question-4-longest-trip)
- [Question 5: Biggest Pickup Zone](#question-5-biggest-pickup-zone)
- [Question 6: Largest Tip](#question-6-largest-tip)
- [Question 7: Terraform Workflow](#question-7-terraform-workflow)

---

## Question 1: Understanding Docker Images

**Task:** Run docker with the `python:3.13` image using `bash` as entrypoint and find the pip version.

### Options:
- 25.3 ✅
- 24.3.1
- 24.2.1
- 23.3.1

### Solution

```bash
# Download and run the image
docker run -it --rm --entrypoint bash python:3.13

# Check pip version
pip --version
```

**Answer:** `25.3`

---

## Question 2: Docker Networking & Compose

**Task:** Given the following `docker-compose.yaml`, determine the hostname and port for pgAdmin to connect to PostgreSQL.

### Docker Compose Configuration

```yaml
services:
  db:
    container_name: postgres
    image: postgres:17-alpine
    environment:
      POSTGRES_USER: 'postgres'
      POSTGRES_PASSWORD: 'postgres'
      POSTGRES_DB: 'ny_taxi'
    ports:
      - '5433:5432'
    volumes:
      - vol-pgdata:/var/lib/postgresql/data

  pgadmin:
    container_name: pgadmin
    image: dpage/pgadmin4:latest
    environment:
      PGADMIN_DEFAULT_EMAIL: "pgadmin@pgadmin.com"
      PGADMIN_DEFAULT_PASSWORD: "pgadmin"
    ports:
      - "8080:80"
    volumes:
      - vol-pgadmin_data:/var/lib/pgadmin

volumes:
  vol-pgdata:
    name: vol-pgdata
  vol-pgadmin_data:
    name: vol-pgadmin_data
```

### Options:
- postgres:5433
- localhost:5432
- db:5433
- postgres:5432
- db:5432 ✅

**Answer:** `db:5432`

> **Note:** Within the Docker network, services communicate using service names (not container names) and internal ports.

---

## 📊 Data Preparation

Download the required datasets:

### Green Taxi Trips Data (November 2025)
```bash
wget https://d37ci6vzurychx.cloudfront.net/trip-data/green_tripdata_2025-11.parquet
```
### Green Taxi Trips Data (November 2025) Pipeline 
```
import pandas as pd
from sqlalchemy import create_engine
from tqdm.auto import tqdm
import click
import requests


url = r'https://github.com/DataTalksClub/nyc-tlc-data/releases/download/misc/taxi_zone_lookup.csv'
pg_user = 'root'
pg_pass = 'root'
pg_host = 'localhost'
pg_port = '5432'
pg_db = 'ny_taxi'
year = 2025
month = 11

chunksize = 100000
target_table = 'Zone'
engine = create_engine(f'postgresql://{pg_user}:{pg_pass}@{pg_host}:{pg_port}/{pg_db}')


def get_file_size(url: str) -> int:
    """Get the file size from URL headers."""
    try:
        response = requests.head(url, allow_redirects=True)
        return int(response.headers.get('content-length', 0))
    except:
        return 0


def ingest_data(
        url: str,
        engine,
        target_table: str,
        chunksize: int = 100000,
) -> dict:
    """
    Ingest CSV data from URL into SQL database in chunks with progress tracking.
    
    Returns:
        dict: Statistics about the ingestion (total_rows, chunks_processed)
    """
    total_rows = 0
    chunks_processed = 0
    
    try:
        print(f"Starting data ingestion from: {url}")
        print(f"Target table: {target_table}")
        print(f"Chunk size: {chunksize:,} rows")
        print("-" * 60)
        
        # Get file size for progress bar
        file_size = get_file_size(url)
        if file_size > 0:
            print(f"File size: {file_size / (1024*1024):.2f} MB")
        
        # Create iterator
        df_iter = pd.read_csv(url, iterator=True, chunksize=chunksize)
        
        # Create table from first chunk
        print("\n📋 Creating table schema...")
        first_chunk = next(df_iter)
        first_chunk.head(0).to_sql(
            name=target_table,
            con=engine,
            if_exists="replace",
            index=False
        )
        print(f"✓ Table '{target_table}' created with {len(first_chunk.columns)} columns")
        
        # Insert first chunk
        print(f"\n📥 Inserting data...")
        first_chunk.to_sql(
            name=target_table,
            con=engine,
            if_exists="append",
            index=False
        )
        total_rows += len(first_chunk)
        chunks_processed += 1
        
        # Progress bar for remaining chunks
        pbar = tqdm(
            df_iter,
            desc="Progress",
            unit="chunk",
            bar_format="{l_bar}{bar}| {n_fmt}/{total_fmt} chunks [{elapsed}<{remaining}]"
        )
        
        # Insert remaining chunks with progress
        for df_chunk in pbar:
            df_chunk.to_sql(
                name=target_table,
                con=engine,
                if_exists="append",
                index=False
            )
            total_rows += len(df_chunk)
            chunks_processed += 1
            
            # Update progress description with row count
            pbar.set_postfix({"rows": f"{total_rows:,}"})
        
        pbar.close()
        
        print("\n" + "=" * 60)
        print(f"✓ Ingestion complete!")
        print(f"  • Total rows inserted: {total_rows:,}")
        print(f"  • Chunks processed: {chunks_processed}")
        print(f"  • Table: {target_table}")
        print("=" * 60)
        
        return {
            "total_rows": total_rows,
            "chunks_processed": chunks_processed,
            "target_table": target_table
        }
        
    except Exception as e:
        print(f"\n✗ Error during ingestion: {str(e)}")
        raise


if __name__ == "__main__":
    # Run the ingestion
    stats = ingest_data(
        url=url,
        engine=engine,
        target_table=target_table,
        chunksize=chunksize
    )`
```

### Taxi Zone Lookup Data
```bash
wget https://github.com/DataTalksClub/nyc-tlc-data/releases/download/misc/taxi_zone_lookup.csv
```
### Taxi Zone Lookup Data (Pipeline)
```
import pandas as pd
from sqlalchemy import create_engine
from tqdm.auto import tqdm
import requests
import tempfile
import os


url = r'https://d37ci6vzurychx.cloudfront.net/trip-data/green_tripdata_2025-11.parquet'
pg_user = 'root'
pg_pass = 'root'
pg_host = 'localhost'
pg_port = '5432'
pg_db = 'ny_taxi'
year = 2025
month = 11

chunksize = 100000
target_table = 'green_tripdata'
engine = create_engine(f'postgresql://{pg_user}:{pg_pass}@{pg_host}:{pg_port}/{pg_db}')


def get_file_size(url: str) -> int:
    """Get the file size from URL headers."""
    try:
        response = requests.head(url, allow_redirects=True)
        return int(response.headers.get('content-length', 0))
    except:
        return 0


def download_file(url: str, local_path: str) -> None:
    """Download file from URL with progress bar."""
    file_size = get_file_size(url)
    
    response = requests.get(url, stream=True)
    response.raise_for_status()
    
    with open(local_path, 'wb') as f:
        if file_size > 0:
            pbar = tqdm(
                total=file_size,
                unit='B',
                unit_scale=True,
                desc='Downloading'
            )
            for chunk in response.iter_content(chunk_size=8192):
                f.write(chunk)
                pbar.update(len(chunk))
            pbar.close()
        else:
            for chunk in response.iter_content(chunk_size=8192):
                f.write(chunk)


def ingest_parquet_data(
        url: str,
        engine,
        target_table: str,
        chunksize: int = 100000,
) -> dict:
    """
    Ingest Parquet data from URL into SQL database in chunks with progress tracking.
    
    Returns:
        dict: Statistics about the ingestion (total_rows, chunks_processed)
    """
    total_rows = 0
    chunks_processed = 0
    temp_file = None
    
    try:
        print(f"Starting data ingestion from: {url}")
        print(f"Target table: {target_table}")
        print(f"Chunk size: {chunksize:,} rows")
        print("-" * 60)
        
        # Get file size
        file_size = get_file_size(url)
        if file_size > 0:
            print(f"File size: {file_size / (1024*1024):.2f} MB")
        
        # Download parquet file to temp location
        print("\n📥 Downloading Parquet file...")
        temp_file = tempfile.NamedTemporaryFile(delete=False, suffix='.parquet')
        download_file(url, temp_file.name)
        print(f"✓ Downloaded to: {temp_file.name}")
        
        # Read parquet file
        print("\n📖 Reading Parquet file...")
        df = pd.read_parquet(temp_file.name)
        print(f"✓ Loaded {len(df):,} rows with {len(df.columns)} columns")
        
        # Display column info
        print(f"\nColumns: {', '.join(df.columns.tolist())}")
        
        # Create table from schema
        print("\n📋 Creating table schema...")
        df.head(0).to_sql(
            name=target_table,
            con=engine,
            if_exists="replace",
            index=False
        )
        print(f"✓ Table '{target_table}' created")
        
        # Insert data in chunks with progress bar
        print(f"\n📥 Inserting data in chunks...")
        num_chunks = (len(df) + chunksize - 1) // chunksize
        
        pbar = tqdm(
            total=num_chunks,
            desc="Progress",
            unit="chunk",
            bar_format="{l_bar}{bar}| {n_fmt}/{total_fmt} chunks [{elapsed}<{remaining}]"
        )
        
        for start_idx in range(0, len(df), chunksize):
            end_idx = min(start_idx + chunksize, len(df))
            df_chunk = df.iloc[start_idx:end_idx]
            
            df_chunk.to_sql(
                name=target_table,
                con=engine,
                if_exists="append",
                index=False
            )
            
            total_rows += len(df_chunk)
            chunks_processed += 1
            pbar.update(1)
            pbar.set_postfix({"rows": f"{total_rows:,}"})
        
        pbar.close()
        
        print("\n" + "=" * 60)
        print(f"✓ Ingestion complete!")
        print(f"  • Total rows inserted: {total_rows:,}")
        print(f"  • Chunks processed: {chunks_processed}")
        print(f"  • Table: {target_table}")
        print("=" * 60)
        
        return {
            "total_rows": total_rows,
            "chunks_processed": chunks_processed,
            "target_table": target_table
        }
        
    except Exception as e:
        print(f"\n✗ Error during ingestion: {str(e)}")
        raise
    
    finally:
        # Clean up temp file
        if temp_file and os.path.exists(temp_file.name):
            os.unlink(temp_file.name)
            print(f"\n🗑️  Cleaned up temporary file")


if __name__ == "__main__":
    # Run the ingestion
    stats = ingest_parquet_data(
        url=url,
        engine=engine,
        target_table=target_table,
        chunksize=chunksize
    )
```

---

## Question 3: Counting Short Trips

**Task:** For trips in November 2025, count how many had a `trip_distance` ≤ 1 mile.

**Date Range:** `lpep_pickup_datetime` between `2025-11-01` and `2025-12-01` (exclusive upper bound)

### Options:
- 7,853
- 8,007 ✅
- 8,254
- 8,421

### SQL Query

```sql
SELECT COUNT(*)
FROM green_tripdata
WHERE trip_distance <= 1
  AND lpep_pickup_datetime >= '2025-11-01'
  AND lpep_pickup_datetime < '2025-12-01';
```

**Answer:** `8,007`

---

## Question 4: Longest Trip

**Task:** Find the pickup day with the longest trip distance (excluding distances ≥ 100 miles to filter out errors).

### Options:
- 2025-11-14 ✅
- 2025-11-20
- 2025-11-23
- 2025-11-25

### SQL Query

```sql
SELECT 
    trip_distance, 
    lpep_pickup_datetime
FROM green_tripdata
WHERE trip_distance < 100
ORDER BY trip_distance DESC
LIMIT 1;
```

**Answer:** `2025-11-14`

---

## Question 5: Biggest Pickup Zone

**Task:** Find the pickup zone with the largest `total_amount` (sum of all trips) on November 18th, 2025.

### Options:
- East Harlem North ✅
- East Harlem South
- Morningside Heights
- Forest Hills

### SQL Query

```sql
SELECT 
    SUM(A.total_amount) AS total_amount,
    B."Zone"
FROM public.green_tripdata A
LEFT JOIN public."Zone" B 
    ON B."LocationID" = A."PULocationID"
WHERE lpep_pickup_datetime >= '2025-11-18' 
    AND lpep_pickup_datetime < '2025-11-19'
GROUP BY B."Zone"
ORDER BY SUM(A.total_amount) DESC
LIMIT 1;
```

**Answer:** `East Harlem North`

---

## Question 6: Largest Tip

**Task:** For passengers picked up in "East Harlem North" during November 2025, find the drop-off zone with the largest tip.

### Options:
- JFK Airport
- Yorkville West ✅
- East Harlem North
- LaGuardia Airport

### SQL Query

```sql
SELECT 
    A.tip_amount,
    B."Zone" AS drop_off_zone
FROM public.green_tripdata A
LEFT JOIN public."Zone" B 
    ON B."LocationID" = A."DOLocationID"
WHERE A."PULocationID" = 74  -- East Harlem North
    AND A.lpep_pickup_datetime >= '2025-11-01'
    AND A.lpep_pickup_datetime < '2025-12-01'
ORDER BY A.tip_amount DESC
LIMIT 1;
```

**Answer:** `Yorkville West`

---

## 🔧 Terraform

### Setup

In your environment (VM/Laptop/Codespace), install Terraform and copy the course repository files to create GCP resources (Bucket and BigQuery Dataset).

## Question 7: Terraform Workflow

**Task:** Identify the correct sequence for:
1. Downloading provider plugins and setting up backend
2. Generating proposed changes and auto-executing the plan
3. Removing all resources managed by Terraform

### Options:
- `terraform import`, `terraform apply -y`, `terraform destroy`
- `terraform init`, `terraform plan -auto-apply`, `terraform rm`
- `terraform init`, `terraform run -auto-approve`, `terraform destroy`
- `terraform init`, `terraform apply -auto-approve`, `terraform destroy` ✅
- `terraform import`, `terraform apply -y`, `terraform rm`

### Explanation

```bash
# 1. Initialize Terraform (download providers, setup backend)
terraform init

# 2. Apply changes automatically without manual approval
terraform apply -auto-approve

# 3. Destroy all managed resources
terraform destroy
```

**Answer:** `terraform init, terraform apply -auto-approve, terraform destroy`

---

## 📚 Key Takeaways

- Docker networking uses service names for inter-container communication
- SQL window functions and aggregations are essential for data analysis
- Terraform follows a clear workflow: `init` → `plan/apply` → `destroy`
- Always filter out data anomalies (like trips > 100 miles) for accurate analysis

---

## 🤝 Contributing

Feel free to open issues or submit pull requests if you find any errors or have suggestions for improvements!

---

## 📄 License

This homework is part of the DataTalks.Club Data Engineering course.

---

**Made with ❤️ for learning Docker, SQL, and Terraform**
