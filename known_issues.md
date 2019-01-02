AWS Glue issues
1. input_file_name() issues
glueContext.create_dynamic_frame_from_options().toDF().withColumn("filepath", input_file_name()) won't work in Glue. It always passes Null value in column. It is known issue and AWS Glue team is aware of it.
