# Spark Declarative Pipelines and Lakeflow Designer on Databricks

1. Orientation, Setup, and Dataset Exploration
    1. [Course Overview & What You Will Build](/docs/1-1-course-overview.md)
    1. [Getting Started in Databricks and Exploring the Dataset](/docs/1-2-dataset-exploration.md)
1. Delta Lake & Unity Catalog Foundations
    1. [Delta Lake Essentials (ACID, Time Travel, Versioning)](/docs/2-1-delta-lake-essentials.md)
    1. [Unity Catalog Basics](/docs/2-2-unity-catalog-basics.md)
1. Building Declarative Pipelines With Pipeline Editor
    1. [Introduction to Declarative Pipelines & Medallion Architecture](/docs/3-1-introduction-declarative-pipelines.md)
    1. [Building the Bronze Layer](/docs/3-2-building-bronze-sql.md) _Alternative*: [Building the Bronze Layer w/ Python](/docs/3-3-building-bronze.md)_
    1. [Building the Silver Layer (Cleaning & Modeling)](/docs/3-3-building-silver-sql.md)
        _Alternative*: [Building the Silver Layer (Cleaning & Modeling) w/ Python](/docs/3-3-building-silver.md)_
    1. [Building the Gold Layer (Business Aggregations)](/docs/3-4-building-gold-sql.md)
    _Alternative*: [Building the Gold Layer (Business Aggregations) w/ Python](/docs/3-3-building-bronze.md)_
1. Orchestrating Declarative Pipelines with Lakeflow Designer
    1. [Introduction to Lakeflow Designer](/docs/4-1-introduction-lakeflow-designer.md)
    1. [Scheduling and Running the Pipeline in Lakeflow Designer](/docs/4-2-scheduling-and-running-pipeline.md)
    1. [Updating the Pipeline Inside Lakeflow Designer](/docs/4-3-updating-pipeline-lakeflow-designer.md)
1. Bonus: Streaming with Declarative Pipelines in Lakeflow Designer
    1. [Streaming Modes in Declarative Pipelines](/docs/5-1-streaming-modes.md)
    1. [Connecting Databricks to an AWS Message Queue](/docs/5-2-connecting-aws.md)
    1. [Building a Simple Streaming Pipeline in Lakeflow Designer](/docs/5-3-streaming-pipeline.md)
1. Summary & Next Steps
    1. [Summary & Next Steps](/docs/6-1-summary.md)

\* For 3.2 to 3.4 I got a version with Python. But I'm not sure if this will work with Lakeflow Designer. Therefore, I rebuilt it using SQL which should work the same. Python can, therefore, most likely be ignored.

