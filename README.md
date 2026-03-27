Ingestion:
    Raw data was ingested from azure blob storage using fivetran connection. A connection string was given to form a connection. Data was ingested from 6 different CSV files and was mapped to 6 different tables at                azure_blob_storage which is inside deminiproject catalog.
    
Transformation:
    Raw data which was ingested from the last step is cleaned and transformed for business usage. All the notebooks are stored in the silver layer of the project. Six different notebooks each for it's respective tables.          Transformation includes changing all date formats to YYYY-MM-DD, standardizing column names to snake case, removing metadata added by fivtran, handling null values and removing special characters from customer names.         After all the transformation the tables were stored in the silver layer of deminiproject catalog.

Gold Layer:
    First two notebooks were created, sales and inventory which are used to create their respective fact tables along with the dim tables. After the dim and fact tables are created and stored, another notebook named kpi is       created and used for calculating kpi which can be used to make business decisions. One final notebook is created named datacube which creates and displays two datacubes cretaed from fact_inventory and fact_sales tables       respectively.

Orhcestration:
    For orchestration databricks' jobs were used. Since the silver layer notebooks are transforming different tables and are independent of each other they are parallel to each other. After these tasks are completed sales and
    inventory notebook tasks run which are again parallel to each other but dependent on the completion of the previous 5 tasks of silver layer. Once all this is done kpi task runs and in serial manner datacube task.
