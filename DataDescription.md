## Database Description

#### Data source
This dataset is an ensemble of data collected from TMDB and GroupLens. The Movie Details, Credits and Keywords have been collected from the TMDB Open API. 

Please be reminded that the TMDb API used in this project is not endorsed or certified by TMDb. Their API also provides access to data on many additional movies, actors and actresses, crew members, and TV shows. You can try it for yourself here. The Movie Links and Ratings have been obtained from the Official GroupLens website. The files are a part of the dataset available here.

#### Why choose this data 
Datasets collected by GroupLens are filtered semi-structured dataset which provides rich information in any period that we want. After due diligence the website, we conclude that GroupLens is a valid website and provide credible, reliable, organized, data. To obtain relevant movie data from GroupLens is convenient for us to conduct further analysis and to use for visualization. 
  
#### Sample of our dataset 
For the display,  we split the 22 columns of data into 8 pieces, since each row is too long due to the column number. Since this is only a sample of our total dataset, we only selected one row as an example. 

| id  | adult    |belongs_to_collection |budget |
| --- | ---      |  ---                 | ---   |
| 862 | FALSE |{'id': 10194, 'name': 'Toy Story Collection', 'poster_path': '/7G9915LfUQ2lVfwMEEhDsn3kT4B.jpg', 'backdrop_path': '/9FBwqcd9IRruEDUrTdcaafOMKUq.jpg'} |3.00E+07 |


#### Plan for analyzing the data
Machine learning models like boosting and random forest to predict future revenue of the upcoming movie based on the past movie performance.
Clustering analysis and sentimental analysis for what may contribute to the success of a movie.

#### Visualization
We will create interfaces using Metabase to visualize the performance of our movie performance with update, which will also allow options for our clients to DIY our data without query the database directly.

#### Weekly analytic report
We will come up with our prediction as well as our recommendation in the weekly analytic report for the business decision.
Please click the link below to see the ER-diagram of our database: ER-Diagram.
Please click the link below to see the code to create our database: SQL CODE.

#### Normalization Plan
Our data comes in an unnormalized form, with JSON objects in some of the column. We have designed a normalization plan to transform it from unnormalized dataset to 3NF. After the normalization process, we have 16 tables in 3NF. 


##### Unnormalized to 1NF
The original table contains many JSON objects in a column, which makes the original table not atomic, such as “belongs_to_collection” and “genres”. We take each of the column out to make them individual tables. Each of these tables have id inherited, so we add the id back in the original table. We name our modified unnormalized table as “user_movie_rating”, and we also have 8 new tables taken from 8 columns separately, which are “collections”, “genres”, “spoken_languages”, “production_companies”, “production_countries”, “crew”, “cast”, and “keywords”. Here we reach 1NF.

##### 1NF to 2NF
The 8 new added tables have reached 2NF. But for the “user_movie_rating” table, the new added id columns like “collection_id” etc are not fully dependent on the key, and they are making the “user_movie_rating” table cumbersome. So we decide to take them out, and each of them combine with the “movie_id” to form relational tables that store the relationship between each element with movies. Specifically, we form the “movies_genres” table that has 2 columns, one is “movie_id”, and the other is “genre_id”. These tables connect two tables with a many to many relation. After seperating id columns from the “user_movie_rating” table, and we change the name of  “user_movie_rating” table into “movies”, which we expect to contain columns that only relate to “movie_id”. We now have 19 tables and we reach 2NF.

##### 2NF to 3NF
19 tables above except “movies” table have reached 3NF. In the “movies” table, “vote_average” and “vote_count” columns are partly dependent on the “tmdbId” column, which is transitive functional dependency of non-prime attribute. Hence we take the “vote_average”, “vote_count” and “tmdbId” out to form a new table named “tmdb_votes”, and we also create the relation table “tmdb_link” to connect votes and movies. We also create a table named “imdb_link” to connect the “movies” and “imdb”. So we reach 22 tables and in our 3NF.

There are 22 tables that we will transform from the original dataset. Generally, there are 10 tables are paired between the table movies and other tables, and the rest of the tables are quite independent. We use RStudio to process and upload our data.

Before the transformation, we notice that there are 0 value in the column “budget” and “revenue”. Although it will not influence the result of our transformation, it may affect the analysis we will come up in the future, so we drop rows containing 0 value in “budget” or “revenue” at the beginning. Since then, we will not drop any of our data in the original dataset. After going through the data, we decide to separate the table <movies> from the original dataset first, as it contains the basic information of movies and has the most foreign key constraint. 

Then we separate the table that do not require further process and can be selected directly from the original dataset. They are <users>, <tmdb_votes>, <tmdb_links>, <imdb_links> and <grouplens_ratings>. We will rename some of the attributes to match the attribute names in the database we created in PgAdmin in this process.In constructing the table <grouplens_ratings>, we typically change the data type of the variable “timestamp”, as it is in integer format, and we should transform into timestamp so to match the data type we have created in the database.

The remaining 16 tables can be processed in pairs, as there are many many to many relation in this database. In each pair, we have 1 table storing basic information of a character, and another table storing the relation between the id of the character and the id of movies.  Specifically, all 8 pairs of the tables are in JSON format in the original database, and we can take the “movie_id” and the certain column out, forming a new dataframe and then transform into 2 new tables. 

For example, as for genre, the table <movies_genres> contains attributes “movie_id” and “genre_id”, while the table <genres> contains attributes “genre_id” and “genre_name”. We then select the columns “movie_id” and “genres” from the original dataset and make it a new data frame named “genredf”. Then we use the “jsonlite” package to unnest the manipulate the data frame into the format we want. We will also go through renaming the column in this process. We now have “genre_id”, “movie_id” and “genre_name” in the “genredf”. Then we will select the distinct pair of “genre_id” and “movie_id” to form a new table named <movies_genres>, and select distinct “genre_id” together with the “genre_name” to form another new table named <genres>. So we have the 2 tables containing the data we want we want through this process.

However, the problems we face when transforming the JSON object into pairs of tables is much more complicated. Specifically, when we deal with the column “collections”, we found that this column is not strictly following the JSON rules, so the package “jsonlite” doesn’t work at first. Then we come up with the idea to add a pair of brackets before and after each row of “collection”, so to make it work in “jsonlite”. Besides, columns like “casts” and “crews” are much more difficult as they are not following JSON rules and have much more accidence. Like we have to transform most of the single quotes to the double quotes in these 2 columns, but there are some single quotes like J’s in the name that we should not make it double quotes. Here we come up with our solution that is very tricky to solve this question by looking for only the combined symbols from and after the single quotes we want to replace, and quotes we don’t want to replace will leave unattacked. For more details please go to our R code. 

Last but not least, we connect to the PgAdmin and upload all the data we have in the RStudio to the PgAdmin. The SQL code for normalization please refer to the Appendix







