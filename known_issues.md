AWS Glue issues
1. input_file_name() issues
  a. glueContext.create_dynamic_frame_from_options().toDF().withColumn("filepath", input_file_name()) won't work in Glue. It always passes Null value in column. It is known issue and AWS Glue team is aware of it.
  b. glueContext.create_dynamic_frame.from_catalog().toDF().withColumn("filepath", input_file_name()) works in Glue. But it means you should first run crawler over data and create Glue Catalogue in order to use this method. This method can't be use directly over data in S3 bucket.
  c. input_file_name() method loads files from S3 bucket to Glue EC2 harddisk for reading file names. Glue EC2 has 45GB hard-disk per node. If data crosses 45GB, Glue will throw 'exit status: -100. diagnostics: container released on a *lost* node' error. This can't be fixed even increasing DPUs. There is no indication in Glue metrics or Glue logs when such scenario happens. Overall usage of input_file_name()  method has performance bottleneck too.
  
2.Auto Schema detection in Glue
Glue recommends to use Glue read/write APIs in order to detect schema correctly. If developer usage pyspark read API and Glue write API, schema will not be detected correctly and all columns will have String datatype.

3. PySpark write API issues
In Glue pyspark write API usage one executor to write data (Glue metrics shows one executor only). Hence write operation takes more time and increase in DPUs doesn't improve performance.

Glue recommendation is to use Glue read/write APIs along with "groupFiles": 'inPartition' flag in order to distribute data evenly among different clusters and to perform operations in parallel.

Reference: https://docs.aws.amazon.com/glue/latest/dg/monitor-profile-debug-straggler.html 

4. Glue Metrics Limitations
Glue shows metrics for executors,  RAM memory and CPU usages. But it doesn't show details about stages, tasks and hard-disk usage. Stages/Tasks details are required in order to understand parallelism used during execution. Hard-disk usage is required in case data is moved from S3 to Glue EC2 hard-disk. 

5. Reading CSV data 
  a. com.amazonaws.services.glue.util.FatalException: Unable to parse file
    i. Glue works with UTF-8 encoded data. If it detects ASCII encoded data, Glue throughs error ' com.amazonaws.services.glue.util.FatalException: Unable to parse file: ' 
    I used below command to convert from ASCII to UTF-8. We can use any text editor(Sublime text) to convert the same.
    - iconv -f CP1250 -t UTF-8 < {input-file-ASCII} > {output-file-UTF-8}
    ii. Pyspark reads data even if it is ASCII coded and there is no issue while using Pyspark read API.
  b. UnicodeEncodeError: 'ascii' codec can't encode character u'\xfc' in position 769: ordinal not in range(128)
  Error may occurs when toDF() method is used after converting ASCII to UTF-8. Following code should fix it:
  import codecs
  sys.stdout = codecs.getwriter('utf8')(sys.stdout)
  c. Crawling over CSV data with ~ separator
  Glue Crawler doesn't detect ~ separator and returns delimiter as 'Unknown'. However Glue Read APIs will able to parse data and create dynamicframe.
  Workarounds:
    a. Use Glue Read API to read csv files and convert them in parquet files before running crawler.  
    b. Use classifier to add custom Grok pattern and then run crawler.  It has 2 limitations:
        i. One has to specify entire schema in Grok pattern. Any change in schema (column names, order of columns) means classifier Grok pattern must be updated.
        ii. Crawler over custom classifier doesnt work with gzip files. Hence gzip files must be unarchived to csv files and then crawler with custom classifier should be run over csv files.

6. Pushdown Predicate Limitations
  a. It works with partition columns only
  b. It works when data is kept in S3 bucket only
  


