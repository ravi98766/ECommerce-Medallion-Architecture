***Link: https://github.com/ravi98766/ECommerce-Medallion-Architecture***
## Project Overview:ECommerce-Medallion-Architecture
***End-to-End Data Pipeline Using Fabric, GitHub API, and Lakehouse Architecture**
This project implements an automated, parameter-driven data ingestion and transformation pipeline using Microsoft Fabric. 
The objective is to extract data hosted in a public GitHub repository, land it in a ShoppingMart Bronze Lakehouse, refine it in the Silver layer, and produce analytics-ready Delta tables in the Gold layer for BI reporting.

## 1. Ingestion Framework and TaskFlow Orchestration

***1.1 TaskFlow***
<img width="1366" height="768" alt="1Data ingest" src="https://github.com/user-attachments/assets/5fa65d7f-5d53-437c-a2bb-8b3667c0ca23" />
<img width="1366" height="768" alt="2Bronze layer" src="https://github.com/user-attachments/assets/c955d7cb-19d5-4703-9a53-ec287092528f" />
<img width="1366" height="768" alt="3further process notebook" src="https://github.com/user-attachments/assets/17d943d6-1d3f-40c5-a99f-2b0c5c453074" />
<img width="1366" height="768" alt="4silver layer" src="https://github.com/user-attachments/assets/29262db8-74b8-4515-88f6-033ca92342f1" />
<img width="1366" height="768" alt="5 furthe gold transformation" src="https://github.com/user-attachments/assets/d53db80e-1b1a-48c1-b863-4c72ebc16489" />
<img width="1366" height="768" alt="6 gold layer" src="https://github.com/user-attachments/assets/913e6063-08c9-43a2-82a9-e0c98c9f09b8" />


***1.1 Parameter-Driven Lookup Design***
To achieve dynamic ingestion, a central lookup JSON file was created containing the following parameters:

•	***source_url***: Fully qualified GitHub API endpoint for the raw dataset.

•	***sink_folder***: Target folder inside the Fabric Lakehouse.

•	***sink_file***: Target filename for ingestion.

Separate lookup JSONs were maintained for:

•	***Structured data (CSV)***
<img width="510" height="280" alt="METADATA FILE FOR CSV" src="https://github.com/user-attachments/assets/d393bf59-9f61-4314-a8e9-d7e0455e3971" />

•	***Unstructured data (JSON)***
<img width="526" height="266" alt="METADATA FILE FOR JSON" src="https://github.com/user-attachments/assets/7bff26ce-5332-430b-aa9e-bd454fc9ecf8" />

These files enable consistent, reusable ingestion logic.

***1.2 Copying Lookup Metadata into Fabric***

A Copy Activity loads both structured and unstructured lookup JSON files into Fabric storage. This establishes the metadata foundation needed for downstream orchestration.

***1.3 TaskFlow and Activity Orchestration***

A Fabric Taskflow was designed where:

•	Lookup Activity reads each record from the metadata file.

•	ForEach Activity iterates through every dataset’s configuration.

•	Inside the loop:

o	Delete Activity clears existing folder contents (ensuring idempotent runs).

o	Copy Activity fetches the actual GitHub file and writes it to the specified sink_folder and sink_file.

This creates a scalable ingestion pattern where any new file can be added by simply updating the lookup JSON.

___________________________________________________________________________________
## 2. Bronze Layer (Raw Zone)
<img width="1366" height="768" alt="1bronze layer lakehouse" src="https://github.com/user-attachments/assets/31ee1dc0-3c87-4624-ba75-d01cd1eed1fb" />

In the ShoppingMart Bronze Lakehouse:

•	Raw files fetched from GitHub API are stored as-is.

•	Files are maintained in their native formats—CSV or JSON.

•	Bronze acts as the immutable landing zone for audit and traceability.
<img width="1366" height="768" alt="4Pipeline 1 Structured" src="https://github.com/user-attachments/assets/66913554-6adf-4c6b-9f7e-85c3dfca1c3d" />
<img width="1366" height="768" alt="5Pipeline 1 Structured details" src="https://github.com/user-attachments/assets/d9ed953f-3292-41cb-8cfa-0f50a2cb2a71" />
<img width="1366" height="768" alt="6Pipeline 1 UnStructured" src="https://github.com/user-attachments/assets/0aab767e-d5ad-4c54-9876-9fb4164fada7" />
<img width="1366" height="768" alt="7Pipeline 1 UnStructured Details" src="https://github.com/user-attachments/assets/6f06af11-ddf7-44dc-b2bb-8d5fb2f2ed04" />

_____________________________________________________________________________________
## 3. Silver Layer (Cleansed, Conformed Zone)

***3.1 Lakehouse Setup***

A dedicated ShoppingMart Silver Lakehouse was created.

***3.2 Shortcut-Based Access***

Instead of copying data again, the Bronze data is exposed into the Silver layer using Fabric Shortcuts, enabling secure, optimized access without duplication.

***3.3 Data Cleaning with PySpark***

A Fabric Notebook (PySpark) performs:

•	Schema enforcement

•	Type casting

•	Date parsing

•	Null handling

•	Removing duplicates

•	Standardizing column formats

•	Converting cleaned data to:

o	Parquet files

o	Delta tables

## LOADING BRONZE DATA INTO SILVER LAYER LAKEHOUSE 

```bash
from pyspark.sql.functions import *
```
```bash
#Loading CSV files

df_customers = spark.read.format("csv") \
    .option("header", "true") \
    .load("Files/ShoppingMart_Bronze_Customers/ShoppingMart_customers.csv")

df_products = spark.read.format("csv") \
    .option("header", "true") \
    .load("Files/ShoppingMart_Bronze_Products/ShoppingMart_products.csv")

df_orders = spark.read.format("csv") \
    .option("header", "true") \
    .load("Files/ShoppingMart_Bronze_Orders/ShoppingMart_orders.csv")

#Loading JSON files
df_reviews = spark.read.json("Files/ShoppingMart_Bronze_Reviews/ShoppingMart_review.json")

df_socialmedia = spark.read.json("Files/ShoppingMart_Bronze_Social_Media/ShoppingMart_social_media.json")

df_weblogs = spark.read.json("Files/ShoppingMart_Bronze_Web_Logs/ShoppingMart_web_logs.json")
```
```bash
df_orders = (
    df_orders
    .dropDuplicates(subset=["CustomerID","ProductID"])
    .withColumn(
        "OrderDate",
        to_date(col("OrderDate"))
    )
    .withColumn("Quantity", col("Quantity").cast("int"))
    .withColumn("TotalAmount", col("TotalAmount").cast("int"))
    .dropna(subset=["OrderID","CustomerID","ProductID","OrderDate"])
)
```
```bash
df_products = (
    df_products
        .dropDuplicates(subset=["ProductID"])
        .withColumn("Stock", col("Stock").cast("int"))
        .withColumn("UnitPrice", col("UnitPrice").cast("int"))
        .dropna(subset=["ProductID"])
)
```
```bash
df_customers = (
    df_customers
    .dropDuplicates(subset=["CustomerID"])
    .withColumn(
        "SignupDate",
        to_date(col("SignupDate"))
    )
    .dropna(subset=["CustomerID"])
)
```
```bash
df_orders = (
    df_orders
        .join(df_products, on="ProductID", how="inner")
        .join(df_customers, on="CustomerID", how="inner")
)
```
```bash
df_reviews = (
    df_reviews
    .dropDuplicates(subset=["product_id"])
    .dropna(subset=["product_id"])
)

display(df_reviews)
```
```bash
df_weblogs = (
   df_weblogs
    .dropDuplicates(subset=["user_id","page","action"])
    .dropna(subset=["user_id","page","action"])
)

display(df_weblogs)
```
```bash
df_orders.write.format("parquet").mode("overwrite").save("Files/ShoppingMart_silverOrders")

df_reviews.write.format("parquet").mode("overwrite").save("Files/ShoppingMart_silverreviews")

df_socialmedia.write.format("parquet").mode("overwrite").save("Files/ShoppingMart_silversocialmedia")

df_weblogs.write.format("parquet").mode("overwrite").save("Files/ShoppingMart_silverweblogs")
```
```bash
df_orders.write.format("delta").mode("overwrite").saveAsTable("silverOrders")

df_reviews.write.format("delta").mode("overwrite").saveAsTable("silverreviews")

df_socialmedia.write.format("delta").mode("overwrite").saveAsTable("_silversocialmedia")

df_weblogs.write.format("delta").mode("overwrite").saveAsTable("silverweblogs")
```



The Silver zone serves as the harmonized, analytics-ready layer.

<img width="1366" height="768" alt="2silver layer lakehouse" src="https://github.com/user-attachments/assets/95325c99-031b-4d69-9b50-6a973aeb3e0f" />

_________________________________________________________________________________________________________________
## 4. Gold Layer (Presentation Zone for BI)

<img width="1366" height="768" alt="3goldlayer lakehouse" src="https://github.com/user-attachments/assets/bad46171-a56d-48f0-a879-568f5e9fb29f" />

A ShoppingMart Gold Lakehouse is created to host business-level curated datasets.

***4.1 Shortcut from Silver***

Gold uses shortcuts to reference Silver Delta tables efficiently.

***4.2 Aggregations and Business Transformations***

Using Fabric Notebooks and SQL, final aggregated Delta tables were built, such as:

•	Category-level and product-level sales metrics

•	Customer insights

•	Revenue and profitability KPIs

•	Time-series summaries

•	Any additional custom business views needed by BI consumers

## LOADING SILVER LAYER DATA FOR GOLD LAYER TRANSFORMATION
```bash
from pyspark.sql.functions import *
```

```bash
df_orders = spark.read.parquet("Files/ShoppingMart_silverOrders/part-00000-9aea8ad2-e70e-46a6-861f-cdf724f02f10-c000.snappy.parquet")
df_reviews = spark.read.parquet("Files/ShoppingMart_silverreviews/part-00000-81b381b7-d541-4d8d-8995-baab9eff1fce-c000.snappy.parquet")
df_socialmedia = spark.read.parquet("Files/ShoppingMart_silversocialmedia/part-00000-1c168628-b1df-479e-9c73-2aa9ccff1bf1-c000.snappy.parquet")
df_weblogs = spark.read.parquet("Files/ShoppingMart_silverweblogs/part-00000-542faaf7-6082-43c4-b97b-2bb605ff6c0b-c000.snappy.parquet")
```
```bash
df_orders.write.format("delta").mode("overwrite").saveAsTable("OrdersGold")
```
```bash
df_reviews=df_reviews.groupby("product_id").agg(avg("rating").alias("AvgRating"))
df_reviews.write.format("delta").mode("overwrite").saveAsTable("reviewsGold")
```
```bash
df_socialmedia=df_socialmedia.groupby("platform","sentiment").count()
df_socialmedia.write.format("delta").mode("overwrite").saveAsTable("socialmediaGold")
```
```bash
df_weblogs=df_weblogs.groupby("user_id","page","action").count()
df_weblogs.write.format("delta").mode("overwrite").saveAsTable("weblogsGold")
```


***4.3 BI-Ready Output***

Final Delta tables are structured for direct consumption by:

•	Power BI semantic models
________________________________________________________________________________________________________________________
## 5. Final Pipeline Orchestration and BI Reporting

<img width="891" height="168" alt="8final pipeline" src="https://github.com/user-attachments/assets/41f45bbb-65ac-445e-85ac-9b518f7c27f7" />

•	Fully automated ingestion from GitHub using dynamic metadata.

•	Clear separation of Bronze, Silver, and Gold Lakehouse layers aligned with medallion architecture.

•	Reusable parameter-driven pipeline enabling easy expansion as more GitHub datasets are added.

•	Clean, validated Silver data with PySpark.

•	Aggregated Gold data ensuring BI readiness and performance efficiency.

•	Dashboards

•	Additional downstream analytics

<img width="620" height="506" alt="Screenshot (1215)" src="https://github.com/user-attachments/assets/b3c5eebd-568a-4eb8-b685-06260a447a6a" />




______________________________________________________________________________________________________


