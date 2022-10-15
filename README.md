# Amazon_Vine_Analysis_Big_Data_Processing
# Overview
## Purpose 
Amazon Vine is a program that allows manufacturers and publishers to receive reviews for their products from members who are paid to be a part of this program. Companies can have access to Amazon Vine reviews by paying only a small fee to Amazon and providing products to Amazon Vine members. The task for this project was to access the Vine reviews of US video games and determine if there was any bias towards favorable reviews from the Vine members in the dataset. 
## Analysis Roadmap 
The review dataset used in this project was for (reviews on US Video Games)[https://s3.amazonaws.com/amazon-reviews-pds/tsv/amazon_reviews_us_Video_Games_v1_00.tsv.gz].  To answer any questions about the dataset it first had to be extracted, transformed, and loaded into a database before it could be queried. Amazon’s Relational Database Service was used to create a database instance that could be accessed and updated in any location via pgAdmin through the RDS server. PySpark was used in Google CoLaboratory to extract, transform, and load data into the database. After, PySpark was used in pgAdmin to filter the data and determine if there is any bias toward favorable reviews from Vine members in the dataset.

This project was broken down into the following two parts found in the Analysis Section:
1. Performing (E)xtract-(T)ransform-(L)oad on the Amazon Vine Product Reviews
   - Create a database with Amazon RDS and connect to it via pgAdmin
   - Create the following four tables for the database in pgAdmin
     1. customers_table
     2. products_table
     3. review_id_table
     4. vine_table
   - Extract the video games review dataset and load it into a DataFrame with PySpark in Google CoLab
   - Transform the extracted DataFrame into four DataFrames that match the schema in the pgAdmin tables 
   - Load the DataFrames into pgAdmin
2. Determining Bias of Vine Reviews 
   - Filter the data and create a new DataFrame to pick reviews that are more likely to be helpful 
   - Filter the DataFrame with helpful reviews and create a new DataFrame to retrieve rows where the majority of votes were helpful
   - Filter the DataFrame where the majority of votes were helpful and create a new DataFrame to retrieve all rows where the review written was part of the paid Vine program.
   - Filter the DataFrame where the majority of votes were helpful and create a new DataFrame to retrieve all rows where the review written was not part of the paid Vine program
  
The Results Section will provide a bulleted list of answers to the following questions:
1. How many Vine reviews and non-Vine reviews were there?
2. How many Vine reviews were 5 stars? How many non-Vine reviews were 5 stars?
3. What percentage of Vine reviews were 5 stars? What percentage of non-Vine reviews were 5 stars?
  
Finally, the Summary Section will dicuss any conclusions made about positivity bias for reviews from the Vine program. There will also be a suggestion for an additional analysis that could be performed on the dataser to suppport the conclusions. 
# Analysis
## Performing ETL on the Amazon Vine US Video Games Reviews 
### Database Creation 
An AWS RDS database was created and connected to pgAmin where four tables were added following the schema below:

```
-- Review ID table
CREATE TABLE review_id_table (
  review_id TEXT PRIMARY KEY NOT NULL,
  customer_id INTEGER,
  product_id TEXT,
  product_parent INTEGER,
  review_date DATE -- yyyy-mm-dd format
);
-- Products table--Unique values only
CREATE TABLE products_table (
  product_id TEXT PRIMARY KEY NOT NULL UNIQUE,
  product_title TEXT
);
-- Customer table 
CREATE TABLE customers_table (
  customer_id INT PRIMARY KEY NOT NULL UNIQUE,
  customer_count INT
);
-- Vine Review table
CREATE TABLE vine_table (
  review_id TEXT PRIMARY KEY,
  star_rating INTEGER,
  helpful_votes INTEGER,
  total_votes INTEGER,
  vine TEXT,
  verified_purchase TEXT
);
```
### Performing ETL on the Amazon Vine Product Reviews
The code for the ETL process was written using Google CoLab so that I could access the Apache Spark analytics engine for its flexibility in big data processing. I accessed Spark through its Python API, PySpark, so that I could write the code for the ETL process in Python instead of Scala. The code referenced in the sections below can be found in the Resources folder in this repository as [an .ipynb file](Resources/Amazon_Reviews_ETL.ipynb) exported from CoLab.
#### Data Extraction 
The [dataset file] (https://s3.amazonaws.com/amazon-reviews-pds/tsv/amazon_reviews_us_Video_Games_v1_00.tsv.gz) can be found in Amazon’s file storage service, S3. This dataset was extracted and loaded into a Spark DataFrame using the following code:
```
from pyspark import SparkFiles
url = "https://s3.amazonaws.com/amazon-reviews-pds/tsv/amazon_reviews_us_Video_Games_v1_00.tsv.gz"
spark.sparkContext.addFile(url)
df = spark.read.option("encoding", "UTF-8").csv(SparkFiles.get(""), sep="\t", header=True, inferSchema=True)
```
This code generated a DataFrame with the following schemata:

![df_schemata](https://user-images.githubusercontent.com/104794100/195958950-49666020-1280-44da-9061-e2bbb6351f8f.png)

#### Data Transformation 
The original DataFrame was transformed into four separate DataFrames that matched the schema in the pgAdmin tables.
##### customers_table DataFrame
To create the customers_table DataFrame, I grouped by the “customer_id” column, aggregated the counts of each customer id, then renamed the counts column to match the schema for the “customers_table” in pgAdmin.
```
customers_df = df.groupby("customer_id").agg({"customer_id": "count"}).withColumnRenamed("count(customer_id)", "customer_count")
```
This code generated the DataFrame in the snapshot below:

![customers_table_sample](https://user-images.githubusercontent.com/104794100/195959092-d2ffb06b-e042-43ab-8cce-324b507af5c9.png)

##### products_table DataFrame
To create the products_table DataFrame, I selected for two columns from the original DataFrame and dropped any duplicate values to retrieve only unique values.
```
products_df = df.select(["product_id", "product_title"]).drop_duplicates()
```
This code generated the DataFrame in the snapshot below:

![products_table_sample](https://user-images.githubusercontent.com/104794100/195959206-abb3a473-f1fc-4332-a1ce-c3e04951ba20.png)

##### review_id DataFrame
To create the review_id DataFrame, I selected for five columns from the original DataFrame and converted the values of the “review_date” column to retrieve only the date in yyyy-MM-dd format.
```
from pyspark.sql.functions import to_date
review_id_df = df.select(["review_id", "customer_id", "product_id", "product_parent", to_date("review_date", 'yyyy-MM-dd').alias("review_date")])
```
This code generated the DataFrame in the snapshot below:

![review_id_sample](https://user-images.githubusercontent.com/104794100/195959250-99124896-2b12-4f52-9a21-e3f95016d4a5.png)

##### vine_table DataFrame
To create the vine_table DataFrame, I selected for six columns from the original DataFrame.
```
vine_df = df.select(["review_id", "star_rating", "helpful_votes", "total_votes", 
                     "vine", "verified_purchase"])
```
This code generated the DataFrame in the snapshot below:

![vine_table_sample](https://user-images.githubusercontent.com/104794100/195959274-bdbe2321-6233-4154-8fd4-ff7d825fa126.png)

#### Data Loading 
I first established a connection to my AWS RSD instance. Then, I loaded the DataFrames into their corresponding tables in pgAdmin. I used the code below to perform the data loading step except I removed my database password for security purposes.
```
# Configure settings for RDS
mode = "append"
jdbc_url="jdbc:postgresql://videogamereviews.cqbjx59yzhab.us-east-2.rds.amazonaws.com:5432/VideoGameReviews"
config = {"user":"postgres", 
          "password": "<password>", 
          "driver":"org.postgresql.Driver"}
# Write review_id_df to table in RDS
review_id_df.write.jdbc(url=jdbc_url, table='review_id_table', mode=mode, properties=config)
# Write products_df to table in RDS
products_df.write.jdbc(url=jdbc_url, table='products_table', mode=mode, properties=config)
# Write customers_df to table in RDS
customers_df.write.jdbc(url=jdbc_url, table='customers_table', mode=mode, properties=config)
# Write vine_df to table in RDS
vine_df.write.jdbc(url=jdbc_url, table='vine_table', mode=mode, properties=config)
```
Once this code was run in Google CoLab, I turned to pgAdmin to query the database and check that the data was properly loaded. 
### Determining Bias of Vine Reviews
The code for the following transformations was written using Google CoLab in PySpark (the reasoning can be found above in the ETL Section). The code referenced in the sections below can be found in the Resources folder in this repository as [an .ipynb file](Resources/Vine_Review_Analysis.ipynb) exported from CoLab. 
#### Filter the data and create a new DataFrame to pick reviews that are more likely to be helpful
This DataFrame was created by transforming the vine_table DataFrame. This new DataFrame was named twenty_plus_votes_df, and it was created to retrieve rows where the reviews ("review_id") recieved more than 20 votes ("total_votes").
```
twenty_plus_votes_df = vine_df.filter(vine_df['total_votes'] >= 20)
```
This code generated the DataFrame in the snapshot below:

![Screen Shot 2022-10-15 at 1 47 14 AM](https://user-images.githubusercontent.com/104794100/195971172-1962926a-89b3-4067-8da8-b76ec87cf6e5.png)

#### Filter the DataFrame with helpful reviews and create a new DataFrame to retrieve rows where the majority of votes were helpful
This DataFrame named was created by transforming twenty_plus_votes_df. This new DataFrame was named fifty_percent_df, and it was created to retrieve rows where the number of "helpful_votes" divided by the number of "total_votes" is greater than or equal to 50%.
```
fifty_percent_df = twenty_plus_votes_df.filter(twenty_plus_votes_df["helpful_votes"]/twenty_plus_votes_df["total_votes"]>0.5)
```
This code generated the DataFrame in the snapshot below:

![Screen Shot 2022-10-15 at 1 54 27 AM](https://user-images.githubusercontent.com/104794100/195971391-f928869e-8c5e-4314-ab5d-609a2b9abb25.png)

#### Filter the DataFrame where the majority of votes were helpful and create a new DataFrame to retrieve all rows where the review written was part of the paid Vine program.
This DataFrame was created by transforming fifty_percent_df. This new DataFrame was named vine_review_df, and it was created to retreive rows where the the values in the "vine" column was equal to "Y".
```
vine_review_df = fifty_percent_df.filter(fifty_percent_df['vine']== 'Y')
```
This code generated the DataFrame in the snapshot below:

![Screen Shot 2022-10-15 at 2 01 56 AM](https://user-images.githubusercontent.com/104794100/195971651-ca254ee3-3768-440b-93e8-659d8db423f3.png)

#### Filter the DataFrame where the majority of votes were helpful and create a new DataFrame to retrieve all rows where the review written was not part of the paid Vine program.
This DataFrame was created by transforming fifty_percent_df. This new DataFrame was named not_vine_review_df, and it was created to retreive rows where the the values in the "vine" column was equal to "N".
```
not_vine_review_df = fifty_percent_df.filter(fifty_percent_df['vine']== 'N')
```
This code generated the DataFrame in the snapshot below:

![Screen Shot 2022-10-15 at 2 02 06 AM](https://user-images.githubusercontent.com/104794100/195971657-a98be02a-1c77-4e9b-9835-1b5d851f440c.png)

# Results 
The code referenced in the sections below can be found in the Resources folder in this repository as [an .ipynb file](Resources/Vine_Review_Analysis.ipynb) exported from CoLab.

- How many Vine reviews and non-Vine reviews were there?
There was a total of **_40,0009 US video game reviews_**
```
vine_review_total = vine_review_df.count()
print(vine_review_total)
not_vine_review_total = not_vine_review_df.count()
print(not_vine_review_total)
print(vine_review_total + not_vine_review_total)
```
Output:
```
94
39915
40009
```
- How many Vine reviews were 5 stars? How many non-Vine reviews were 5 stars?
There was a total of 15,604 5-star reviews. 
There was a total of **_48 5-star reviews from Vine memebers_**.
There was a total of **_15,556 5-star reviews from non-Vine memebers_**.
```
vine_five_stars = vine_review_df.filter(vine_review_df['star_rating']== 5).count()
print(vine_five_stars)
not_vine_five_stars = not_vine_review_df.filter(not_vine_review_df['star_rating']==5).count()
print(not_vine_five_stars)
total_five_stars = (vine_five_stars + not_vine_five_stars)
print(total_five_stars)
```
Output:
```
48
15556
15604
```
- What percentage of Vine reviews were 5 stars? What percentage of non-Vine reviews were 5 stars?
Out of the 94 Vine reviews, **_51% were 5-star reviews_**.
Out of the 39,915 non-Vine reviews, **_39% were 5-star reviews_**.
```
percentage_vine_five_stars = vine_five_stars/vine_review_total 
print(percentage_vine_five_stars)
percentage_not_vine_five_stars = not_vine_five_stars/not_vine_review_total 
print(percentage_not_vine_five_stars)
```
Output:
```
0.5106382978723404
0.3897281723662783
```
# Summary 
Based on the results above, I conclude that there is positivity bias for reviews in the Vine program since 51% of the Vine reviews were 5-star reviews whereas only 38% of the non-Vine reviews were 5-star reviews. One should consider the total number of reviews leading to these percentages. The data contained only 94 Vine reviews which could have led to the bias exhibited. 

## Suggestions
I propose further analysis where more data is included for the Vine sample to see if the bias still exists. If a new DataFrame was created to retrieve rows where the reviews ("review_id") recieved more than 10 votes ("total_votes"), this could increase the sample size of Vine reviews and potentially eliminate the positivity bias.

When this proposed analysis was performed, the sample of Vine reviews increased to 204 reviews where 92 of those reviews were 5-star reviews. This means 45% of the Vine reviews were 5-star reviews which is only a slight decrease from the original 51%, so I stand by my conclusion that there is positivity bias for reviews in the Vine program. 

The code for this analysis is shown below:
```
ten_plus_votes_df = vine_df.filter(vine_df['total_votes'] >= 10)
fifty_percent_of_10_df = ten_plus_votes_df.filter(ten_plus_votes_df["helpful_votes"]/twenty_plus_votes_df["total_votes"]>0.5)
expanded_vine_review_df = fifty_percent_of_10_df.filter(fifty_percent_of_10_df['vine']== 'Y')
expanded_vine_review_total = expanded_vine_review_df.count()
print(expanded_vine_review_total)
expanded_vine_five_stars = expanded_vine_review_df.filter(expanded_vine_review_df['star_rating']== 5).count()
print(expanded_vine_five_stars)
percentage_expanded_vine_five_stars = expanded_vine_five_stars/expanded_vine_review_total 
print(percentage_expanded_vine_five_stars)
```
Output:
```
204
92
0.45098039215686275
```


