# 3.4 Building the Gold Layer (Business Aggregations)

Now with cleaned data, we can tackle the last layer of the medallion architecture. The gold layer has the answers to the questions the business has about our data.

Whenver you build a production system, you will start here. Not technically, but with the questions or problems you need to solve. Then you start building your pipelines around that.

For this course, we are going to build a single gold aggregation that we can then visualize. The data we want to have is how many sales did we have per day.

## Create a Gold Transformation File

As a starting point, we go back to Databricks and jump again into 'Jobs & Pipelines'.

![Jobs & Pipelines menu item](/images/3-4/gold-jobs-and-pipelines.png)

Again, select our 'online_retail_pipeline':

![Online retail pipeline in the list](/images/3-4/gold-online-retail-pipeline.png)

Now on the top select 'Edit pipeline' to open the pipeline editor.

![Edit pipeline button in the online retail pipeline graph](/images/3-4/gold-edit-pipeline.png)

This will open a new tab with the editor. Right click on the folder '02_gold'. Then select 'Create file'.

![Create file dialog on Gold folder](/images/3-4/create-gold-transformation-file.png)

Again, we will keep Python. For the name, we are going to use 'daily_sales'. Lastly, for the dataset type, we go with a materialized view this time.

> [!NOTE]
> Sadly, at the time of writing the materialized view type still generates a streaming table. Let's hope they will fix it. You can also use no template and just go on.

We use materialized views for Gold aggregations because they are optimized for aggregations. Materialized views cache pre-computed results, making dashboard queries extremely fast. They can often use automatic incremental updates, so when new orders arrive, Databricks only recomputes affected dates and not the entire history. It handles late data correctly and in our case I assume there is no version history needed. Business metrics just need current accuracy, not audit trails of every change.

![Gold file input with name, type, and language](/images/3-4/create-file-input.png)

## Build The Aggregation Logic With Databricks Assistant

Let's start with the basic syntax to read from the silver streaming table.

```python
from pyspark import pipelines as dp

@dp.materialized_view(
    name="retail_pipeline.gold.daily_sales"
)
def daily_sales():
    return spark.read.table("retail_pipeline.silver.silver_orders")
```

You have several options here. You can now either go step by step or try to use one prompt with a lot of context and fine-tune it later. Since we already went step-by-step in Silver, let's try to use Databricks Assistant this time to get a lot right from the beginning and then do some minor tweaks.

Use this prompt via (`CMD` + `I`):
```
Add to the existing daily_sales materialized view operators to group the data by date (convert invoice_date to date). 

Exclude cancellations (where is_cancellation = true).

Aggregate these metrics:
- total_orders: distinct count of invoice numbers (invoice_no)
- total_revenue: sum of total_price
- unique_customers: distinct count of customers (customer_id)
- total_items_sold: sum of quantity
```

For me, that created a separate method and some suboptimal code. But we can improve it from here. Accept the result first.

![Databricks Assistant first result](/images/3-4/databricks-assistant-daily-sales.png)

You should now have this incorrect code:

```python
from pyspark import pipelines as dp

@dp.materialized_view(
    name="retail_pipeline.gold.daily_sales"
)
def daily_sales():
    return spark.readStream.table("retail_pipeline.silver.silver_orders")
from pyspark.sql.functions import col, to_date, sum as _sum, countDistinct

def daily_sales():
    df = spark.read.table("retail_pipeline.silver.silver_orders")
    return (
        df.filter(~col("is_cancellation"))
          .groupBy(to_date(col("invoice_date")).alias("date"))
          .agg(
              countDistinct("invoice_no").alias("total_orders"),
              _sum("total_price").alias("total_revenue"),
              countDistinct("customer_id").alias("unique_customers"),
              _sum("quantity").alias("total_items_sold")
          )
    )
```

Let's first move the imports to the top and remove the obsolete daily_sales function definition.

We should end up with this:

```python
from pyspark import pipelines as dp
from pyspark.sql.functions import col, to_date, sum as _sum, countDistinct

@dp.materialized_view(
    name="retail_pipeline.gold.daily_sales"
)
def daily_sales():
    return (
        spark.read.table("retail_pipeline.silver.silver_orders")
            .filter(~col("is_cancellation"))
            .groupBy(to_date(col("invoice_date")).alias("date"))
            .agg(
                countDistinct("invoice_no").alias("total_orders"),
                _sum("total_price").alias("total_revenue"),
                countDistinct("customer_id").alias("unique_customers"),
                _sum("quantity").alias("total_items_sold")
            )
    )
```

Now, we can still add a comment to explain our function a bit.

```python
from pyspark import pipelines as dp
from pyspark.sql.functions import col, to_date, sum as _sum, countDistinct

@dp.materialized_view(
    name="retail_pipeline.gold.daily_sales"
)
def daily_sales():
    """
    Daily sales metrics data mart for business reporting.
    
    Aggregates completed orders (excludes cancellations) by date to provide:
    - Total unique orders per day
    - Total revenue generated
    - Number of unique customers who ordered
    - Total quantity of items sold
    
    Materialized view automatically refreshes when upstream Silver data changes.
    Pre-computed for fast dashboard queries.
    """
    return (
        spark.read.table("retail_pipeline.silver.silver_orders")
            .filter(~col("is_cancellation"))
            .groupBy(to_date(col("invoice_date")).alias("date"))
            .agg(
                countDistinct("invoice_no").alias("total_orders"),
                _sum("total_price").alias("total_revenue"),
                countDistinct("customer_id").alias("unique_customers"),
                _sum("quantity").alias("total_items_sold")
            )
    )
```

Now let's run the pipeline by either clicking on "Run pipeline" or with `CMD` + `Enter` (preferred clicking since results seemed to be better).

This should lead to a successful pipeline run again.

![Successful gold transformation daily sales](/images/3-4/successful-gold-transformation.png)

## Inspect The Result

Let us again investigate the created view in the catalog. I showed you before how to do it but I want to also share a different way with you.

Click on the catalog tab next to the files and folders:

![Catalog tab in file explorer](/images/3-4/catalog-tab-file-explorer.png)

Then expand the retail_pipeline accordingly. You may need to use the refresh button here but then you should see in the gold layer the materialized view.

![Expand the pipeline to see the gold view](/images/3-4/expand-pipeline.png)

Now we already know the materialized view was correctly created. But you can also hover over daily_sales and use the three dots menu and use 'Open in Catalog Explorer' to again open a new tab of the catalog.

![Dot menu after hovering the gold view](/images/3-4/dot-menu-to-catalog.png)

In this tab, I want to show you quickly one additional cool AI feature that we skipped so far.

You get a table or view description from AI which helps less technical users understand the data and helps with discoverability. You see here how Databricks recommends a description which is quite good, so let's accept it:

![AI Description of Databricks for our daily_sales view](/images/3-4/ai-description.png)

You can do the same for our Silver and Bronze tables if you want.

## Build a Quick Visualization

To finish this medallion part off and answer the question of daily sales for our hypothetical stakeholders, we will do a quick visualization of the data.

Jump into the 'SQL Editor'.

![SQL Editor menu item](/images/3-4/sql-editor.png)

Select the workspace (e.g. use `CTRL` + `Option` + `E` or press the button).

![Switch to workspace](/images/3-4/open-workspace.png)

Then use the dot menu (Folder actions) to create a new query.

![Folder actions to create a new query](/images/3-4/folder-actions.png)

Rename the query by right-clicking the previous name to `daily_sales_query`.

Then add this SQL query to our data.

```sql
SELECT * 
FROM retail_pipeline.gold.daily_sales
ORDER BY date ASC;
```

Then hit `CMD` + `Enter` to run the query.

If you get a prompt, simply select 'Start, attach and run'.

![Start, attach and run option](/images/3-4/start-attach-run.png)

Wait until the query finished running with success.

![Successful query](/images/3-4/successful-query.png)

Now as previously, select the '+' icon to add a visualization.

![Add visualization button](/images/3-4/add-visualization-button.png)

Then set the basics for the chart. First make sure that you turn off "New charts" since it wasn't working at all in my tests.
Also select the Visualization type 'Line', set the X column as `date` and the Y column `total_revenue` and `Sum` as the aggregation.

![Basic chart configuration](/images/3-4/basic-chart-settings.png)

Now select the 'X axis' tab and set the name to `Date`.

![X-Axis configuration](/images/3-4/x-axis.png)

Then select the tab 'Y axis' and set the name to `Total Revenue`.

![Y-Axis configuration](/images/3-4/y-axis.png)

Then hit the 'Save' button to save your visualization.

![Save button to save the visualization](/images/3-4/gold-save-chart.png)

Now you should see a finished visualization tab and can let stakeholders run that query.

![Finished visualization of our data](/images/3-4/gold-finished-chart.png)

Obviously, in production setting, you will probably use a more sophisticated visualization solution like Dashboards in Databricks, but for this course this nicely sums up our medallion architecture with the answer to the question how our sales behaved by day. You can obviously add more visualizations or even new gold layer use cases - the data is there.