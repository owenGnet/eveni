
<p align="center">
  <img src="http://oweng.net/Images/eveni_logo_large.png" alt="eveni logo">
</p>

# eveni POC

## the why

Below is the Spark 2.3.1 documentation related to populating a
DataFrame/DataSet via a JDBC connection:

https://spark.apache.org/docs/2.3.1/api/java/org/apache/spark/sql/DataFrameReader.html#jdbc-java.lang.String-java.lang.String-java.lang.String-long-long-int-java.util.Properties-


    org.apache.spark.sql
    Class DataFrameReader

    Object
        org.apache.spark.sql.DataFrameReader
    ...
    Interface used to load a Dataset from external storage systems (e.g. file systems, key-value stores, etc). Use SparkSession.read to access this.

--

    jdbc

    public Dataset<Row> jdbc(String url,
                             String table,
                             String columnName,
                             long lowerBound,
                             long upperBound,
                             int numPartitions,
                             java.util.Properties connectionProperties)

    Construct a DataFrame representing the database table accessible via JDBC URL url named table. Partitions of the table will be retrieved in parallel based on the parameters passed to this function.

    Don't create too many partitions in parallel on a large cluster; otherwise Spark might crash your external database systems.

    Parameters:
        url - JDBC database url of the form jdbc:subprotocol:subname.
        table - Name of the table in the external database.
        columnName - the name of a column of integral type that will be used for partitioning.
        lowerBound - the minimum value of columnName used to decide partition stride.
        upperBound - the maximum value of columnName used to decide partition stride.
        numPartitions - the number of partitions. This, along with lowerBound (inclusive), upperBound (exclusive), form partition strides for generated WHERE clause expressions used to split the column columnName evenly. When the input is less than 1, the number is set to 1.
        connectionProperties - JDBC database connection arguments, a list of arbitrary string tag/value. Normally at least a "user" and "password" property should be included. "fetchsize" can be used to control the number of rows per fetch.
    Returns:
        (undocumented)
    Since:
        1.4.0


The problem set <font color='#7C7C7C' style="font-family:consolas,monospace;font-size:125%" >eveni</font>
is aiming to help out with relates selecting a
column and number of partitions for the `columnName` and `numPartitions`
parameter in above by querying the source table via Python.
Advising on most efficient  `lowerBound` and `upperBound`
values is a possible future goal.

Simply put, there may be several candidate columns within a given source
table that might serve as a partitioning key but the effectiveness and
efficiency of the partioning operations will be reliant on the actual
distribution of values across that column.


Let's assume a table (`some_table`) exists that includes a column named `some_column` and
that the relevant parameters are set as below

    column_name = 'some_column'
    lower_bound = 0
    upper_bound = 100
    num_partitions = 10

This would produce code like below in Spark, with a given executor
executing one or more of the queries, depending on number and availability
of executors:

        SELECT * FROM some_table.some_column < 10 OR some_column IS NULL
        SELECT * FROM some_table.some_column >= 20 AND some_column < 30
        SELECT * FROM some_table.some_column >= 30 AND some_column < 40
        SELECT * FROM some_table.some_column >= 40 AND some_column < 50
        SELECT * FROM some_table.some_column >= 50 AND some_column < 60
        SELECT * FROM some_table.some_column >= 60 AND some_column < 70
        SELECT * FROM some_table.some_column >= 70 AND some_column < 80
        SELECT * FROM some_table.some_column >= 80 AND some_column < 90
        SELECT * FROM some_table.some_column >= 90

In a best case scenario each one of those queries would return the
same number of rows and roughly the same amount of actual data, i.e.
one query range wouldn't have 10 times as many rows as the others combined. The
latter part (amount of data) is a bit on the unknowable side for most cases but it
shouldn't be too much work to at least select a column that has a
more even distribution of values vs. other candidate columns in that same
table. If the column being partitioned has a wildly uneven distribution
of values then one of the ranges in the above queries might have one
executor working on it for 60 minutes while the other 9 queries are
quickly completed by other executors. Same amount of data accessed
through a column with an even distribution might take 10 minutes as each
of 10 executors are able to run, and complete, their database reads
fully in parallel.

The goal of <font color='#7C7C7C' style="font-family:consolas,monospace;font-size:125%" >eveni</font>
 is to aid in selecting column that will allow
for greatest efficiency, avoiding the scenario where there are a few Spark
executors lingering behind the others.

## the code

The proof of concept is in a Jupyter notebook and everything should be
relatively straightforward as there are interstitial notes. The first part
involves generating some test data and testing the approach using multiple
database flavors. After that it switches over to more of a simulation mode,
how the code might be used in practice by concentrating on the data retrieved
from one o those databases.

Summary of notebook workflow
1) perform imports, both `pyodbc` and `turbodbc` are used for communicating with db,
`pandas` for data manipulation and `HoloViews`/`bokeh` for visualization
2) create a bunch of fake data, including some columns with predefined distributions
3) insert the above data into each of the db systems being tested
    - Vertica
    - MS SQL Server
    - PostgreSQL
4) pull it all back out again, from each of the databases, into one master DataFrame
5) introspect the defined datatypes for target table in each of the databases
    - for POC only going to consider integer columns as candidates for partitioning
    - other numeric columns can work, and/or a simple expression could be
    used to extract data from non-numeric values, e.g. parse minute values from a datetime column
6) using the scala code (`columnPartition()` in JDBCRelation.scala) for Spark 2.3
as reference, construct python equivalent for generating SQL range queries
7) select one of the databases (Vertica) to proceed further with, create appropriate
filter queries based on actual data within candidate columns
8) run the filter queries in order to see how many rows will be in each
column/partition combo for each of the hard-coded `numPartitions` possibilities
9) come up with measure for ranking evenness of row counts within each bin,
aggregated on a per-column level
10) display visualizations of rows-per-bin distributions across columns and/or
given `numPartitions` counts

| columns in dropdown sorted by lowest average standard deviation | dynamically select different column/numPartitions combination |
| --- | --- |
| ![alt text](http://owenG.net/images/eveni_poc_hv_default_small.png "eveni sample default") | ![alt text](http://owenG.net/images/eveni_poc_hv_selected_small.png "eveni sample selected") |



In addition to making something practically useful, below is scratch list
of some features and scenarios I would want to look into handling:
- test with real-world data
- handle other data type beyond integer, in terms of increasing effort
    - other number datatypes, e.g. floats
    - parse time units from datetime-like columns, e.g. 60 partitions based on
minute parts of 0 through 59
    - custom expressions, anything from the `LENGTH` of a column's values to
some using a regex-like SQL function, e.g. `REGEXP_SUBSTR` in Vertica, to
pull numbers out of a `VARCHAR` field
- advise on `lowerBound` and `upperBound` values, absolute min/max values
may not work best for skewed distributions
- multiple tables within a given schema
- create French Canadian variant, because I really want to use the name **yveseni**™







