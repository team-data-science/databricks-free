# 3.3 Building the Silver Layer (Cleaning & Modeling)

Afer we've built the Bronze layer, we'll continue with Silver. For silver you have several options, but I will again keep it simple and do some basic cleaning steps while maintaining the stucture of the data as it is. This gives us the flexibility to build our use cases in Gold on top of that.

## Create a Silver Transformation File

Let's get back to our pipeline by selecting 'Jobs & Pipelines'.

![Jobs & Pipelines menu item](/images/3-2/open-jobs-and-pipelines.png)

Then select 'online_retail_pipeline'.

![Our previously built pipeline](/images/3-3/pipelines.png)

Now click on 'Edit pipeline'.

![Edit pipeline button](/images/3-3/edit-pipeline.png)

In the newly opened browser tab, make sure the workspace is selected.

![Selected workspace tab](/images/3-3/workspace-selected.png)

Now create a new file in your directory '01_silver' by right-clicking on the folder and selecting 'Create file'.

Name the file `clean_orders`, again use Python as the language but this time, we're going to use the template and not start from scratch. So select `Streaming table` as the 'Dataset type'.

![File creation dialog for Silver cleaning](/images/3-3/create-silver.png)

The template starts with a basic streaming table from a path. We, however, want to use our previously created bronze table. It also adds the basic structure, a comment, and a few imports.

Let's first clean our code a bit up to this:

```python
from pyspark import pipelines as dp

@dp.table(
  name="retail_pipeline.silver.silver_orders"
)
def clean_orders():
    """
    Cleans the data by:
    """

    return (
        spark.readStream.table("bronze_orders")
    )
```

## Rename Columns With Databricks Assistant

Then let's rename our fields to use lower snake case for all fields and not mix with upper camel case.

That is a lot of manual coding. Luckily we can use Databricks Assistant to our help. Use this prompt (via `CMD` + `I`):

```
Can you rename all fields from bronze_orders that are upper camel case to lower snake case?
```

Then send the prompt. The result may differ for you but I got this very helpful response:

![Renaming result of Databricks Assistant](/images/3-3/renaming-assistant.png)

But you may notice that it used all columns even the ones that are already lower snake case. 

> [!NOTE]
> Always validate the result of AI-coding assistants and check if they do what you want.

Since the above code is very helpful, I do accept it via `Enter` and remove the two obsolete lines. I will also add this in the comment.

Your code should now look like this:

```python
from pyspark import pipelines as dp

@dp.table(
  name="retail_pipeline.silver.silver_orders"
)
def clean_orders():
    """
    Cleans the data by:
    - renaming columns
    """

    return (
        spark.readStream.table("bronze_orders")
        .withColumnRenamed("InvoiceNo", "invoice_no")
        .withColumnRenamed("StockCode", "stock_code")
        .withColumnRenamed("Description", "description")
        .withColumnRenamed("Quantity", "quantity")
        .withColumnRenamed("InvoiceDate", "invoice_date")
        .withColumnRenamed("UnitPrice", "unit_price")
        .withColumnRenamed("CustomerID", "customer_id")
        .withColumnRenamed("Country", "country")
    )
```

## Handle Type Conversions With Databricks Assistant

That was only a start with Databricks Assistant and cleaning our data.

In the next step, let's try to do our type conversions with Databricks Assistant. You probably know, context is key when it comes to prompts. So let't use this clear instruction (again use `CMD` + `I`):

```
Can you please cast the type of customer_id to string, unit_price to a Decimal that makes sense for a price, and lastly the invoice_date to a timestamp where the format is this: M/d/yy H:mm?
```

This will also directly create what we need:

![Prompt to do the type casting](/images/3-3/type-casting-prompt.png)

> [!WARNING]
In the screenshots below I used the conversion to a date which removed the time. The prompt above is correct, and the code below. Just the pictures may differ from what you would expect.

Accept it, then remove the now obsolete rows that before did the renaming since our casting steps do that now as well, and add a comment above. Then we will also add an import to the top for the `col` function. So your code should look like this:

```python
from pyspark import pipelines as dp
from pyspark.sql.functions import col, to_timestamp

@dp.table(
  name="retail_pipeline.silver.silver_orders"
)
def clean_orders():
    """
    Cleans the data by:
    - renaming columns
    - type casting columns
    """

    return (
        spark.readStream.table("bronze_orders")
        .withColumnRenamed("InvoiceNo", "invoice_no")
        .withColumnRenamed("StockCode", "stock_code")
        .withColumnRenamed("Description", "description")
        .withColumnRenamed("Quantity", "quantity")
        .withColumnRenamed("Country", "country")
        .withColumn("customer_id", col("CustomerID").cast("string"))
        .withColumn("unit_price", col("UnitPrice").cast("decimal(10,2)"))
        .withColumn("invoice_date", to_timestamp(col("InvoiceDate"), "M/d/yy H:mm"))
    )
```

## Add Derived Fields With Databricks Assistant

Now our Silver data is already in a much better shape. Let us now handle cancellations which are important to really understand the performance of products or the sales.

Do you remember the negative quantities we had in our dataset exploration in chapter 1.2? 

We found a lot of products had negative quantities and that these products' invoice numbers started with a 'C' indicating it is a cancellation.

Let's handle that case by add a derived field to indicate if something is a cancellation.

Use the following prompt:

```
For all entries with an invoice_no field that starts with 'C' or 'c' add a column 'is_cancellation' and set the value to true. For all others, set the value to false.
```

This will generate a Regex that helps us handle this case. 

![Regex for cancellation](/images/3-3/regex-cancellation.png)

Accept it and after a little formatting and removing the inline comment and adding a comment on the top, the code should look like this:

```python
from pyspark import pipelines as dp
from pyspark.sql.functions import col, to_timestamp

@dp.table(
  name="retail_pipeline.silver.silver_orders"
)
def clean_orders():
    """
    Cleans the data by:
    - renaming columns
    - type casting columns
    - adding a derived column is_cancellation when invoice_no starts with c or C
    """

    return (
        spark.readStream.table("bronze_orders")
        .withColumnRenamed("InvoiceNo", "invoice_no")
        .withColumnRenamed("StockCode", "stock_code")
        .withColumnRenamed("Description", "description")
        .withColumnRenamed("Quantity", "quantity")
        .withColumnRenamed("Country", "country")
        .withColumn("customer_id", col("CustomerID").cast("string"))
        .withColumn("unit_price", col("UnitPrice").cast("decimal(10,2)"))
        .withColumn("invoice_date", to_timestamp(col("InvoiceDate"), "M/d/yy H:mm"))
        .withColumn("is_cancellation", col("invoice_no").rlike("^[Cc]"))
    )
```

Now let's add one more derived column with the total price. Use this prompt:

```
Can you add a new field 'total_price' that gets calculated by quantity * unit_price?
```

This will generate another field like we defined.

![Prompt for total price](/images/3-3/total-price-prompt.png)

We can accept this as is and add a comment again on the top. Your code should now look like this:

```python
from pyspark import pipelines as dp
from pyspark.sql.functions import col, to_timestamp

@dp.table(
  name="retail_pipeline.silver.silver_orders"
)
def clean_orders():
    """
    Cleans the data by:
    - renaming columns
    - type casting columns
    - adding a derived column is_cancellation when invoice_no starts with c or C
    - adding a derived column total_price as quantity * unit_price
    """

    return (
        spark.readStream.table("bronze_orders")
        .withColumnRenamed("InvoiceNo", "invoice_no")
        .withColumnRenamed("StockCode", "stock_code")
        .withColumnRenamed("Description", "description")
        .withColumnRenamed("Quantity", "quantity")
        .withColumnRenamed("Country", "country")
        .withColumn("customer_id", col("CustomerID").cast("string"))
        .withColumn("unit_price", col("UnitPrice").cast("decimal(10,2)"))
        .withColumn("invoice_date", to_timestamp(col("InvoiceDate"), "M/d/yy H:mm"))
        .withColumn("is_cancellation", col("invoice_no").rlike("^[Cc]"))
        .withColumn("total_price", col("quantity") * col("unit_price"))
    )
```

This already puts our silver data in a good spot. Now it's time for some final quality checks. That is done with Expactions.

## Expectations

Before we add now these quality checks, we first need to understand what Expectations are in Declarative Pipelines. 

![Databricks Expectations flow graph](/images/3-3/expectations-flow-graph.png)

Expactations are an option to apply quality checks directly in Declarative Pipelines. You define an expectation that you have for your data and the pipeline will check against it.

If the check passes, the record will be kept in all cases. However, you can define how a failure should be handled. You have three options:

1. Warn - This will warn you of a record that doesn't match the expecation but will still keep it.
2. Drop - This will simply drop all values that do not match the expectation.
3. Fail Flow - This will fail the complete flow. Imagine that the whole pipeline should fail when a value is NULL or too high for a single record. 

These Expecations also help with monitoring of the quality of your data and can be applied directly to materialized views and streaming tables like the one we created in the Bronze layer. However, the quality metrics will only be available for warnings or dropped values, not for failed flows.

The Expectation syntax consists of standard SQL Boolean statements for the condition and a name. Lastly, you can even reuse them or group them so if one Expectation fails, the records will be handled accordingly to their failure action (warn, drop, or fail flow).

Syntax:
Python
```python
@dp.expect(<constraint-name>, <constraint-clause>)
```
SQL:
```sql
CONSTRAINT <constraint-name> EXPECT ( <constraint-clause> )
```

## Apply Expectations to Ensure Quality

We will add two Expectations.

1. We will make sure that both invoice ID and stock code are not null.
1. Also we will ensure that the price is greater or equal to zero.

It's important to know that the expectation will run on the transformed data and, therefore, you will use the new column names with lower snake case.

You can add more checks later to make the Silver layer more robust on your onw.

The final code should look like this:

```python
from pyspark import pipelines as dp
from pyspark.sql.functions import col, to_timestamp

@dp.table(
  name="retail_pipeline.silver.silver_orders"
)
@dp.expect_or_drop("valid_order", "invoice_no IS NOT NULL AND stock_code IS NOT NULL")
@dp.expect_or_drop("unit_price_non_negative", "unit_price >= 0")
def clean_orders():
    """
    Cleans the data by:
    - renaming columns
    - type casting columns
    - adding a derived column is_cancellation when invoice_no starts with c or C
    - adding a derived column total_price as quantity * unit_price
    - check that both invoice ID and stock code are not null
    - ensure that the unit price is greater than or equal to zero
    """

    return (
        spark.readStream.table("bronze_orders")
        .withColumnRenamed("InvoiceNo", "invoice_no")
        .withColumnRenamed("StockCode", "stock_code")
        .withColumnRenamed("Description", "description")
        .withColumnRenamed("Quantity", "quantity")
        .withColumnRenamed("Country", "country")
        .withColumn("customer_id", col("CustomerID").cast("string"))
        .withColumn("unit_price", col("UnitPrice").cast("decimal(10,2)"))
        .withColumn("invoice_date", to_timestamp(col("InvoiceDate"), "M/d/yy H:mm"))
        .withColumn("is_cancellation", col("invoice_no").rlike("^[Cc]"))
        .withColumn("total_price", col("quantity") * col("unit_price"))
    )
```

We will drop all records that do not match our expectations. Let's run the pipeline (`CMD` + `Enter` or 'Run pipeline' button -> button works better).

![Successful build of silver layer](/images/3-3/successful-run-silver.png)

This will give us a clean silver table with quality checks that run before, correct data types, consistent column names, and some additional derived columns for the gold layer.

## Inspect The Result

In the UI you can immediately see in the pipeline graph the DAG and that silver depends on bronze and that the build was successful.

First let's check our data quality. Click on the 'View Data Quality' button in the bottom as a column of the 'silver_orders'.

![Data quality metrics button](/images/3-3/data-quality-button.png)

This will open the expectations and you can directly see that two records were filtered out because of a negative unit price.

![Data quality metrics](/images/3-3/data-quality-metrics.png)

Currently, on this screen, it only shows the failed Expectations. But if you want to see all expectations, you can select the 'All' tab.

![All expectations shown](/images/3-3/all-expectations.png)

This also shows that all records had a stock code and an invoice ID.

Next let's look into the data in the data catalog. Use the 'View in Catalog' button in the 'silver_orders' row.

![View in Catalog button](/images/3-3/view-in-catalog-button.png)

This will open the catalog in a new browser tab. 

![silver_orders table in catalog](/images/3-3/silver-orders-in-catalog.png)

Here you can see all columns that we've defined with the correct snake case and data types.

You can play around here a bit with the different tabs, like History, Lineage, and Sample Data. But this gives us already what we need for the gold layer use cases. That's what we will build next.
