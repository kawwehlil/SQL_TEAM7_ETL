The dataset is retrieved from Kaggle. The dataset combines the data from IMDB and TMDB open API and GroupLens.
These sources are open to public where we can update our dataset 

There are 22 tables that we will transform from the original dataset. Generally, there are 10 tables are paired between the table movies and other tables, and the rest of the tables are entirely independent. We use RStudio to process and upload our data.

Before the transformation, we notice that there is 0 value in the column “budget” and “revenue.” Although it will not influence the result of our transformation, it may affect the analysis we will come up in the future, so we drop rows containing 0 value in “budget” or “revenue” at the beginning. Since then, we will not drop any of our data in the original dataset.

After going through the data, we decide to separate the table <movies> from the original dataset first, as it contains the necessary information of movies and has the most foreign key constraint.

Then we separate the table that does not require further process and can be selected directly from the original dataset. They are <users>, <tmdb_votes>, <tmdb_links>, <imdb_links> and <grouplens_ratings>. We will rename some of the attributes to match the attribute names in the database we created in PgAdmin in this process. In constructing the table <grouplens_ratings>, we typically change the data type of the variable “timestamp,” as it is in integer format, and we should transform into timestamp so to match the data type we have created in the database.

The remaining 16 tables can be processed in pairs, as there are many many to many relations in this database. In each pair, we have 1 table storing necessary information of a character, and another table storing the relationship between the id of the character and the id of movies.  Specifically, all 8 pairs of the tables are in JSON format in the original database, and we can take the “movie_id” and the certain column out, forming a new data frame and then transform into 2 new tables.

For example, as for genre, the table <movies_genres> contains attributes “movie_id” and “genre_id”, while the table <genres> contains attributes “genre_id” and “genre_name”. We then select the columns “movie_id” and “genres” from the original dataset and make it a new data frame named “genredf.” Then we use the “jsonlite” package to unnest the manipulate the data frame into the format we want. We will also go through renaming the column in this process. We now have “genre_id,” “movie_id” and “genre_name” in the “genredf.” Then we will select the distinct pair of “genre_id” and “movie_id” to form a new table named <movies_genres>, and select distinct “genre_id” together with the “genre_name” to form another new table named <genres>. So we have the 2 tables containing the data we want we want through this process.

However, the problems we face when transforming the JSON object into pairs of tables is much more complicated. Specifically, when we deal with the column “collections,” we found that this column is not strictly following the JSON rules, so the package “jsonlite” doesn’t work at first. Then we come up with the idea to add a pair of brackets before and after each row of “collection,” so to make it work in “jsonlite.” Besides, columns like “casts” and “crews” are much more difficult as they are not following JSON rules and have much more accidence. Like we have to transform most of the single quotes to the double quotes in these 2 columns, but there are some single quotes like J’s in the name that we should not make it double quotes. Here we come up with our solution that is very tricky to solve this question by looking for only the combined symbols from and after the single quotes we want to replace, and quotes we don’t want to succeed will leave unattacked. For more details, please go to our R code.

Last but not least, we connect to the PgAdmin and upload all the data we have in the RStudio to the PgAdmin.

 

Supplementary explanation：

We have made minus adjustments to our schema. Specifically, we change some of the data types and names of the variables for our convenience. If you want to redo our work, please use the updated SQL code below to construct the schema of the database in replace of that provided in the checkpoint 3. All the data are uploaded to the database successfully. Please review our database [Movie] through the username and password provided.

Hostname: s19db.apan5310.com (Links to an external site.)Links to an external site.
Port:     50207
Username: postgres
Password: lfytcyux

Alt- Extraction
------

First, we load the dataset and packages


```r
setwd("~/Documents/2019Spring/APAN 5310 - SQL & RELATIONAL DATABASE/assignment/Final Project/the-movies-dataset")

library(plyr)
library(tidyverse)
library(jsonlite)
library(stringr)
library(dplyr)
library(tidyr)
require('RPostgreSQL')
```

#### Create a connection to the database:
```r
pwd <- 'lfytcyux' 
```

Connect R-studio to Postgre server with RPostgreSQL

```r
con <- dbConnect(drv = dbDriver('PostgreSQL'),
                 dbname = 'Movie',
                 host = 's19db.apan5310.com', port = 50207,
                 user = 'postgres', password = pwd)

data <- read.csv('data.csv')
data <- data[!(data$revenue==0 | data$budget==0),]
colnames(data)[colnames(data)=="movieId"] <- "movie_id"
```

##### movies

```r
movies <- data %>% select(movie_id, original_title, original_language,
                          overview, status, revenue, budget, release_date,
                          runtime, popularity, tagline, video) %>% distinct(movie_id, .keep_all = TRUE)
movies$original_title <- as.character(movies$original_title)
movies[2095,'original_title'] <- 'How I Flew from London to Paris in 25 hours 11 minutes'
```

##### users

```r
users <- data %>% select(userId, adult) %>% distinct(userId, .keep_all = TRUE)
colnames(users)[which(names(users) == "userId")]<-"user_id"
```

##### tmdb_votes

```r
tmdb_votes <- data %>% select(tmdbId, vote_average, vote_count) %>% distinct(tmdbId, .keep_all = TRUE)
colnames(tmdb_votes)[which(names(tmdb_votes) == "tmdbId")]<-"tmdbid"
```

##### tmdb_links

```r
tmdb_links <- data %>% select(movie_id, tmdbId) %>% distinct()
colnames(tmdb_links)[which(names(tmdb_links) == "tmdbId")]<-"tmdbid"
```

##### imdb_links

```r
imdb_links <- data %>% select(movie_id, imdbId) %>% distinct()
colnames(imdb_links)[which(names(imdb_links) == "imdbId")]<-"imdbid"
```

##### GroupLens_ratings
```r
grouplens_ratings <- data %>% select(userId, movie_id, rating, timestamp) %>% distinct(userId, movie_id, .keep_all = TRUE)
colnames(grouplens_ratings)[which(names(grouplens_ratings) == "userId")]<-"user_id"
grouplens_ratings$timestamp <- as.POSIXct(grouplens_ratings$timestamp, origin = "1960-01-01", tz = "GMT")
```

## Transform 

We use R package `jsonlite` to transform JSON objects into tables.

1. The original JSON objects are not in a standard form because of the single quote. We change the single quote into double quote.

##### casts

```r
castdf<-data %>% select(movie_id,cast)
castdf$cast <- as.character(castdf$cast)
castdf$cast <- gsub("\"","\'", castdf$cast)
castdf$cast <- gsub("\', \'","\", \"", castdf$cast,fixed = TRUE)
castdf$cast <- gsub("{\'","{\"", castdf$cast,fixed = TRUE)
castdf$cast <- gsub("\'}","\"}", castdf$cast,fixed = TRUE)
castdf$cast <- gsub("\':","\":", castdf$cast)
castdf$cast <- gsub(": \'",": \"", castdf$cast,fixed = TRUE)
castdf$cast <- gsub(", \'",", \"", castdf$cast)
castdf$cast <- gsub("None","\"None\"", castdf$cast)
castdf$cast <- gsub("\\", "", castdf$cast,fixed = TRUE)
```

2. The `jsonlite` package transforms the JSON column into a table.

```r
castdf= castdf[,c('movie_id','cast')] %>% filter(nchar(cast)>2) %>% mutate(js=lapply(cast,fromJSON)) %>% unnest(js)
castdf$cast <- NULL
castdf$credit_id <- NULL
castdf$cast_id <- NULL
```

3. Rename the column names to match our database and add movie_cast_id for reference.

```r
castdf <- bind_cols('movie_cast_id' = sprintf('ma%07d', 1:nrow(castdf)), castdf)
colnames(castdf)[which(names(castdf) == "id")]<-"cast_id"
colnames(castdf)[which(names(castdf) == "gender")]<-"cast_gender"
colnames(castdf)[which(names(castdf) == "name")]<-"cast_name"
colnames(castdf)[which(names(castdf) == "profile_path")]<-"cast_profile_path"
colnames(castdf)[which(names(castdf) == "order")]<-"cast_order"
movies_casts<-castdf %>% select(movie_cast_id, movie_id, cast_id, character, cast_order) %>% distinct(movie_cast_id, .keep_all = TRUE)
casts<-castdf %>% select(cast_id,cast_name,cast_gender,cast_profile_path) %>% distinct(cast_id, .keep_all = TRUE)
```
4. Select the column we need to join into tables

##### crews
```r
crewdf<-data %>% select(movie_id,crew)
crewdf$crew <- as.character(crewdf$crew)
crewdf$crew <- gsub("\', \'","\", \"", crewdf$crew,fixed = TRUE)
crewdf$crew <- gsub("{\'","{\"", crewdf$crew,fixed = TRUE)
crewdf$crew <- gsub("\'}","\"}", crewdf$crew,fixed = TRUE)
crewdf$crew <- gsub("\':","\":", crewdf$crew)
crewdf$crew <- gsub(": \'",": \"", crewdf$crew,fixed = TRUE)
crewdf$crew <- gsub(", \'",", \"", crewdf$crew)
crewdf$crew <- gsub("\"J.R.\"","J.R.", crewdf$crew,fixed = TRUE)
crewdf$crew <- gsub("\"Gonzo\"","Gonzo", crewdf$crew,fixed = TRUE)
crewdf$crew <- gsub("\"Boomwallah\"","Boomwallah", crewdf$crew,fixed = TRUE)
crewdf$crew <- gsub("\"Cucco\"","Cucco", crewdf$crew,fixed = TRUE)
crewdf$crew <- gsub("None","\"None\"", crewdf$crew)
crewdf= crewdf[,c('movie_id','crew')] %>% filter(nchar(crew)>2) %>% mutate(js=lapply(crew,fromJSON)) %>% unnest(js)
crewdf$crew <- NULL
crewdf$credit_id <- NULL
crewdf <- bind_cols('movie_crew_id' = sprintf('me%07d', 1:nrow(crewdf)), crewdf)
colnames(crewdf)[which(names(crewdf) == "id")]<-"crew_id"
colnames(crewdf)[which(names(crewdf) == "department")]<-"crew_department"
colnames(crewdf)[which(names(crewdf) == "gender")]<-"crew_gender"
colnames(crewdf)[which(names(crewdf) == "job")]<-"crew_job"
colnames(crewdf)[which(names(crewdf) == "name")]<-"crew_name"
colnames(crewdf)[which(names(crewdf) == "profile_path")]<-"crew_profile_path"
movies_crews<-crewdf %>% select(movie_crew_id, movie_id, crew_id, crew_department,crew_job) %>% distinct(movie_crew_id, .keep_all = TRUE)
crews<-crewdf %>% select(crew_id,crew_name,crew_gender,crew_profile_path) %>% distinct(crew_id, .keep_all = TRUE)
```

##### collections
```r
collectiondf= data %>% select(movie_id, belongs_to_collection)
collectiondf$belongs_to_collection <- as.character(collectiondf$belongs_to_collection)
collectiondf$belongs_to_collection <- gsub("None","\"None\"", collectiondf$belongs_to_collection)
collectiondf$belongs_to_collection <- gsub("\', \'","\", \"", collectiondf$belongs_to_collection,fixed = TRUE)
collectiondf$belongs_to_collection <- gsub("{\'","{\"", collectiondf$belongs_to_collection,fixed = TRUE)
collectiondf$belongs_to_collection <- gsub("\'}","\"}", collectiondf$belongs_to_collection,fixed = TRUE)
collectiondf$belongs_to_collection <- gsub("\':","\":", collectiondf$belongs_to_collection)
collectiondf$belongs_to_collection <- gsub(": \'",": \"", collectiondf$belongs_to_collection,fixed = TRUE)
collectiondf$belongs_to_collection <- gsub(", \'",", \"", collectiondf$belongs_to_collection)
collectiondf$belongs_to_collection = paste('[', collectiondf$belongs_to_collection, sep='')
collectiondf$belongs_to_collection = paste(collectiondf$belongs_to_collection, ']',sep='')
collectiondf= collectiondf[,c('movie_id','belongs_to_collection')] %>% filter(nchar(belongs_to_collection)>2) %>% mutate(js=lapply(belongs_to_collection,fromJSON)) %>% unnest(js)
collectiondf$belongs_to_collection <- NULL
colnames(collectiondf)[which(names(collectiondf) == "name")] <- "collection_name"
colnames(collectiondf)[which(names(collectiondf) == "id")] <- "collection_id"
colnames(collectiondf)[which(names(collectiondf) == "poster_path")] <- "collection_poster"
colnames(collectiondf)[which(names(collectiondf) == "backdrop_path")] <- "collection_backdrop_path"
movies_collections <- collectiondf %>% select(movie_id, collection_id) %>% distinct()
collections <- collectiondf %>% select(collection_id, collection_name, collection_poster, collection_backdrop_path) %>% distinct(collection_id, .keep_all = TRUE)
```

##### genres

```r
genredf= data %>% select(movie_id, genres)
genredf$genres <- as.character(genredf$genres)
genredf$genres <- gsub("\'","\"", genredf$genres)
genredf= genredf[,c('movie_id','genres')] %>% filter(nchar(genres)>2) %>% mutate(js=lapply(genres,fromJSON)) %>% unnest(js)
genredf$genres <- NULL
colnames(genredf)[which(names(genredf) == "name")] <- "genre_name"
colnames(genredf)[which(names(genredf) == "id")] <- "genre_id"
movies_genres <- genredf %>% select(movie_id, genre_id) %>% distinct()
genres <- genredf %>% select(genre_id, genre_name) %>% distinct(genre_id, .keep_all = TRUE)
```

##### keywords

```r
keyworddf= data %>% select(movie_id, keywords)
keyworddf$keywords <- as.character(keyworddf$keywords)
keyworddf$keywords <- gsub("\', \'","\", \"", keyworddf$keywords,fixed = TRUE)
keyworddf$keywords <- gsub("{\'","{\"", keyworddf$keywords,fixed = TRUE)
keyworddf$keywords <- gsub("\'}","\"}", keyworddf$keywords,fixed = TRUE)
keyworddf$keywords <- gsub("\':","\":", keyworddf$keywords)
keyworddf$keywords <- gsub(": \'",": \"", keyworddf$keywords,fixed = TRUE)
keyworddf$keywords <- gsub(", \'",", \"", keyworddf$keywords)
keyworddf$keywords <- gsub("None","\"None\"", keyworddf$keywords)
keyworddf$keywords <- gsub("\\", "", keyworddf$keywords,fixed = TRUE)
keyworddf= keyworddf[,c('movie_id','keywords')] %>% filter(nchar(keywords)>2) %>% mutate(js=lapply(keywords,fromJSON)) %>% unnest(js)
keyworddf$keywords <- NULL
colnames(keyworddf)[which(names(keyworddf) == "name")] <- "keyword_name"
colnames(keyworddf)[which(names(keyworddf) == "id")] <- "keyword_id"
movies_keywords <- keyworddf %>% select(movie_id, keyword_id) %>% distinct()
keywords <- keyworddf %>% select(keyword_id, keyword_name) %>% distinct(keyword_id,.keep_all = TRUE)
```

##### spoken_languages

```r
languagedf= data %>% select(movie_id, spoken_languages)
languagedf$spoken_languages <- as.character(languagedf$spoken_languages)
languagedf$spoken_languages <- gsub("\'","\"", languagedf$spoken_languages)
languagedf= languagedf[,c('movie_id','spoken_languages')] %>% filter(nchar(spoken_languages)>2) %>% mutate(js=lapply(spoken_languages,fromJSON)) %>% unnest(js)
languagedf$spoken_languages <- NULL
colnames(languagedf)[which(names(languagedf) == "name")] <- "language_name"
colnames(languagedf)[which(names(languagedf) == "iso_639_1")] <- "language_abbr"
movies_spoken_languages <- languagedf %>% select(movie_id, language_abbr) %>% distinct()
spoken_languages <- languagedf %>% select(language_abbr, language_name) %>% distinct(language_abbr, .keep_all = TRUE)
```

##### production_companies

```r
companydf= data %>% select(movie_id, production_companies)
companydf$production_companies <- as.character(companydf$production_companies)
companydf$production_companies <- gsub("\', \'","\", \"", companydf$production_companies,fixed = TRUE)
companydf$production_companies <- gsub("{\'","{\"", companydf$production_companies,fixed = TRUE)
companydf$production_companies <- gsub("\'}","\"}", companydf$production_companies,fixed = TRUE)
companydf$production_companies <- gsub("\':","\":", companydf$production_companies)
companydf$production_companies <- gsub(": \'",": \"", companydf$production_companies,fixed = TRUE)
companydf$production_companies <- gsub(", \'",", \"", companydf$production_companies)
companydf$production_companies <- gsub("None","\"None\"", companydf$production_companies)
companydf= companydf[,c('movie_id','production_companies')] %>% filter(nchar(production_companies)>2) %>% mutate(js=lapply(production_companies,fromJSON)) %>% unnest(js)
companydf$production_companies <- NULL
colnames(companydf)[which(names(companydf) == "name")] <- "production_company_name"
colnames(companydf)[which(names(companydf) == "id")] <- "production_company_id"
movies_production_companies <- companydf %>% select(movie_id, production_company_id) %>% distinct()
production_companies <- companydf %>% select(production_company_id, production_company_name) %>% distinct(production_company_id, .keep_all = TRUE)
```

##### production_countries

```r
countrydf= data %>% select(movie_id, production_countries)
countrydf$production_countries <- as.character(countrydf$production_countries)
countrydf$production_countries <- gsub("\'","\"", countrydf$production_countries)
countrydf= countrydf[,c('movie_id','production_countries')] %>% filter(nchar(production_countries)>2) %>% mutate(js=lapply(production_countries,fromJSON)) %>% unnest(js)
countrydf$production_countries <- NULL
colnames(countrydf)[which(names(countrydf) == "name")] <- "country_name"
colnames(countrydf)[which(names(countrydf) == "iso_3166_1")] <- "country_abbr"
movies_production_countries <- countrydf %>% select(movie_id, country_abbr) %>% distinct()
production_countries <- countrydf %>% select(country_abbr, country_name) %>% distinct(country_abbr, .keep_all = TRUE)
```

## Load
### Connect the tables and write data in the PgAdmin

1. Use PostgreSQL to create table and store data.
2. Load our created table from R-studio to PostgreSQL.

```r
dbWriteTable(con, name ='movies', value = movies, row.names = FALSE, append = TRUE)
dbWriteTable(con, name ='users', value = users, row.names = FALSE, append = TRUE)
dbWriteTable(con, name ='collections' , value = collections, row.names = FALSE, append = TRUE)
dbWriteTable(con, name ='genres', value = genres, row.names = FALSE, append = TRUE)
dbWriteTable(con, name ='spoken_languages', value = spoken_languages, row.names = FALSE, append = TRUE)
dbWriteTable(con, name ='production_companies', value = production_companies, row.names = FALSE, append = TRUE)
dbWriteTable(con, name ='production_countries', value = production_countries, row.names = FALSE, append = TRUE)
dbWriteTable(con, name ='crews', value = crews, row.names = FALSE, append = TRUE)
dbWriteTable(con, name ='casts', value =casts, row.names = FALSE, append = TRUE)
dbWriteTable(con, name ='keywords', value = keywords, row.names = FALSE, append = TRUE)
dbWriteTable(con, name ='tmdb_votes', value = tmdb_votes, row.names = FALSE, append = TRUE)
dbWriteTable(con, name ='imdb_links', value = imdb_links, row.names = FALSE, append = TRUE)
dbWriteTable(con, name ='grouplens_ratings', value = grouplens_ratings, row.names = FALSE, append = TRUE)
dbWriteTable(con, name ='tmdb_links', value =tmdb_links, row.names = FALSE, append = TRUE)
dbWriteTable(con, name ='movies_keywords', value = movies_keywords, row.names = FALSE, append = TRUE)
dbWriteTable(con, name ='movies_collections', value = movies_collections, row.names = FALSE, append = TRUE)
dbWriteTable(con, name ='movies_genres', value = movies_genres, row.names = FALSE, append = TRUE)
dbWriteTable(con, name ='movies_spoken_languages', value = movies_spoken_languages, row.names = FALSE, append = TRUE)
dbWriteTable(con, name ='movies_production_countries', value = movies_production_countries, row.names = FALSE, append = TRUE)
dbWriteTable(con, name ='movies_production_companies', value =movies_production_companies, row.names = FALSE, append = TRUE)
dbWriteTable(con, name ='movies_casts', value = movies_casts, row.names = FALSE, append = TRUE)
dbWriteTable(con, name ='movies_crews', value = movies_crews, row.names = FALSE, append = TRUE)
```

For other tables, there is no transformation process needed. The dataset can be created by joining the columns we need from the original table, filtering out the duplicated data, and renaming the column.
