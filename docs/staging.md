![alt text](https://raw.githubusercontent.com/Datavault-UK/dbtvault-docs/master/docs/assets/images/staging.png "Staging from a raw table to the raw vault")

>This documentation is based on dbtvault official [example](https://github.com/Datavault-UK/dbtvault-docs/blob/master/docs/worked_example/we_staging.md)

We have two staging layers, as shown in the diagram above.

## The raw staging layer

First we create a raw staging layer. This feeds in data from the source system so that we can process it
more easily. In the `models/raw` folder we have provided two models which set up a raw staging layer.

### raw_orders

The `raw_orders` model feeds data from TPC-H, into a wide table containing all the orders data
for a single day-feed. The day-feed will load data from the day given in the `date` var.

### raw_inventory

The `raw_inventory` model feeds the static inventory from TPC-H. As this data does not contain any dates,
we do not need to do any additional date processing or use the `date` var as we did for the raw orders data.
The inventory consists of the `PARTSUPP`, `SUPPLIER`, `PART` and `LINEITEM` tables.

### raw_transactions

The `raw_inventory` simulates transactions so that we can create Transactional Links. It does this by
making a number of calculations on orders made by customers and creating transaction records.

[Read more](tpch_profile.md#transactions)

## Building the raw staging layer

To build this layer with dbtvault, run the below command:

=== "< dbt v0.20.x"
`dbt run --models tag:raw`

=== "> dbt v0.21.0"
`dbt run -s tag:raw`

Running this command will run all models which have the `raw` tag. We have given the `raw` tag to the
two raw staging layer models, so this will compile and run both models.

The dbt output should give something like this:

```shell
21:43:59  Concurrency: 5 threads (target='dev')
21:43:59  
21:43:59  1 of 3 START sql table model databricks_datavault.raw_orders ................... [RUN]
21:43:59  2 of 3 START sql table model databricks_datavault.raw_transactions ............. [RUN]
21:44:13  2 of 3 OK created sql table model databricks_datavault.raw_transactions ........ [OK in 13.36s]
21:44:13  1 of 3 OK created sql table model databricks_datavault.raw_orders .............. [OK in 13.66s]
21:44:13  3 of 3 START sql table model databricks_datavault.raw_inventory ................ [RUN]
21:44:25  3 of 3 OK created sql table model databricks_datavault.raw_inventory ........... [OK in 12.44s]
21:44:26  
21:44:26  Finished running 3 table models in 0 hours 0 minutes and 32.64 seconds (32.64s).

```

## The 'prepared' staging layer

The tables in the raw staging layer need additional columns to prepare the data for loading to the raw vault.

Specifically, we need to add primary key hashes, hashdiffs, and any implied fixed-value columns
(see the diagram at the top of the page).

We have created a helper macro for dbtvault, to make this step easier; the [stage](https://github.com/Datavault-UK/dbtvault-docs/blob/master/docs/macros/index.md#stage) macro, which
generates derived and hashed columns from a given raw staging table.

### v_stg_orders and v_stg_inventory

The `v_stg_orders` and `v_stg_inventory` models use the raw layer's `raw_orders` and `raw_inventory`
models as sources, respectively. Both are created as views on the raw staging layer, as they are intended as
transformations on the data which already exists.

Each view adds a number of primary keys, hashdiffs and additional constants for use in the raw vault.

### v_stg_transactions

The `v_stg_transactions` model uses the raw layer's `raw_transactions` model as its source

### Using the stage macro

By using the below template and providing the required metadata, the stage macro generates hashed columns, derived columns and automatically selects all columns
from the source table.

```jinja2
{{ dbtvault.stage(include_source_columns=true,
                  source_model=source_model,
                  derived_columns=derived_columns,
                  null_columns=null_columns,
                  hashed_columns=hashed_columns,
                  ranked_columns=none) }}
```

Let's take a look at some metadata supplied to the stage macro for the `v_stg_transactions` view:

=== "v_stg_transactions.sql"

    ```jinja
    {%- set yaml_metadata -%}
    source_model: 'raw_transactions'
    derived_columns:
      RECORD_SOURCE: '!RAW_TRANSACTIONS'
      LOAD_DATE: DATEADD(DAY, 1, TRANSACTION_DATE)
      EFFECTIVE_FROM: 'TRANSACTION_DATE'
    hashed_columns:
      TRANSACTION_HK:
        - 'CUSTOMER_ID'
        - 'TRANSACTION_NUMBER'
      CUSTOMER_HK: 'CUSTOMER_ID'
      ORDER_HK: 'ORDER_ID'
    {%- endset -%}
    
    {% set metadata_dict = fromyaml(yaml_metadata) %}
    
    {% set source_model = metadata_dict['source_model'] %}
    
    {% set derived_columns = metadata_dict['derived_columns'] %}
    
    {% set hashed_columns = metadata_dict['hashed_columns'] %}
    
    {{ dbtvault.stage(include_source_columns=true,
                      source_model=source_model,
                      derived_columns=derived_columns,
                      null_columns=none,
                      hashed_columns=hashed_columns,
                      ranked_columns=none) }}
    ```

#### source_model

The `source_model` defines the raw stage `raw_transactions` which our staging layer will use to extract data from and
prepare the data for the raw vault.

#### derived_columns

We can define any additional inferred, fixed value and calculated columns using the `derived_columns` configuration.

=== "v_stg_transactions.sql"

    ```yaml
    derived_columns:
        RECORD_SOURCE: '!RAW_TRANSACTIONS'
        LOAD_DATE: DATEADD(DAY, 1, TRANSACTION_DATE)
        EFFECTIVE_FROM: 'TRANSACTION_DATE'
    ```

Here, we are creating three new columns, which are all metadata columns
required for auditability in the raw vault.

1. `RECORD_SOURCE` is defined as a constant by prefixing the constant value with `!`. This is syntactic sugar to inform
   dbtvault to treat this value as a constant string and not a column name.

2. `LOAD_DATE` is defined as a calculated column using a function. Here we are synthetically creating a `LOAD_DATE`
   for the purposes of simulating a transaction feed as described earlier in this guide.

3. `EFFECTIVE_FROM` is our business effective date for a given record, and it makes sense to derive this from the `TRANSACTION_DATE`.


#### hashed_columns

Next, we define our `hashed_columns`.

=== "v_stg_transactions.sql"

    ```yaml
    hashed_columns:
      TRANSACTION_HK:
        - 'CUSTOMER_ID'
        - 'TRANSACTION_NUMBER'
      CUSTOMER_HK: 'CUSTOMER_ID'
      ORDER_HK: 'ORDER_ID'
    ```

- We are defining `TRANSACTION_HK` as a new hashed column, which is formed from the concatenation of
  the `CUSTOMER_ID` and `TRANSACTION_NUMBER` columns present in the `raw_transactions` model.

- `CUSTOMER_HK` and `ORDER_HK` are both hashed from single columns, so we provide a single string with the column name.

In the `v_stg_orders` view we can also see an example of a hashdiff column, `CUSTOMER_HASHDIFF`:

=== "v_stg_orders.sql"

    ```yaml
    hashed_columns:
      CUSTOMER_HASHDIFF:
        is_hashdiff: true
        columns:
          - 'CUSTOMERKEY'
          - 'CUSTOMER_NAME'
          - 'CUSTOMER_ADDRESS'
          - 'CUSTOMER_PHONE'
          - 'CUSTOMER_ACCBAL'
          - 'CUSTOMER_MKTSEGMENT'
          - 'CUSTOMER_COMMENT'
    ```

These work very similarly to multi-column hashes (like `TRANSACTION_HK`) except that we provide an `is_hashdiff` flag
with the value `true` and provide the list of columns under a `columns` key.

Defining a hashdiff using this syntax will ensure the columns are automatically alpha-sorted, which is standard practice
for hashdiffs

!!! tip
For more detail on dbtvault's hashing process,
why we hash, and the differences between hash keys and hashdiffs, [read more](https://github.com/Datavault-UK/dbtvault-docs/blob/master/docs/best_practices.md#hashing).

## Deploying the 'prepared' staging layer

To build this layer with dbtvault, run the below command:

=== "< dbt v0.20.x"
`dbt run --models tag:stage`

=== "> dbt v0.21.0"
`dbt run -s tag:stage`

Running this command will run all models which have the `stage` tag. We have given the `stage` tag to the
two hashed staging layer models, so this will compile and run both models.

The dbt output should give something like this:

```shell
21:50:01  Concurrency: 5 threads (target='dev')
21:50:01  
21:50:01  1 of 3 START sql table model databricks_datavault.v_stg_inventory .............. [RUN]
21:50:01  2 of 3 START sql table model databricks_datavault.v_stg_orders ................. [RUN]
21:50:01  3 of 3 START sql table model databricks_datavault.v_stg_transactions ........... [RUN]
21:50:15  3 of 3 OK created sql table model databricks_datavault.v_stg_transactions ...... [OK in 13.75s]
21:50:15  2 of 3 OK created sql table model databricks_datavault.v_stg_orders ............ [OK in 13.84s]
21:50:15  1 of 3 OK created sql table model databricks_datavault.v_stg_inventory ......... [OK in 14.16s]
21:50:16  
21:50:16  Finished running 3 table models in 0 hours 0 minutes and 21.02 seconds (21.02s).

```