
# Cahoot Practical Assessment

## Summary:
* It is my first time creating a PWA, so I might not have done everything correctly
* Since I had experience developing React App, I chose to make an React Progressive Web App
* In order to open the database mdf file, I had to use MSSQL, however, since I have a Macbook, so I had to use the Docker
* I wasted too much time setting up the environment, so I did not have enough time to implemente the push notification
* The page I wrote is in the Page 1 component, the directory is client/src/pages/Page1

## How to run this project:
### Backend:
* cd into the server folder and then run:
```
node app.js
```
### Frontend:
*cd into the client folder and then run (the backend has to be running in order for the frontend to run):
```
npm install
npm run start
```

- I started with this template: https://github.com/suren-atoyan/react-pwa

- I was going to modify the query that I found here https://stackoverflow.com/questions/25483654/sql-query-to-return-the-most-relevant-results-from-a-search
to find the most relevant searches. However, I realized that this operation would not be efficient enought especially when my database is so large. Therefore, 
I had to look for a more efficient method, after some research, I learned about the Microsoft full text search, which generates a Full-Text Index for the table 
which makes full text search a lot more faster. 

## Create the StackOverflow database on Docker 
### Prerequisites 
* Have docker installed 
* Download the stackoverflow database files (both .ldf and .mdf) from [here](https://meta.stackexchange.com/questions/2677/database-schema-documentation-for-the-public-data-dump-and-sede)
* Have this [Dockerfile](https://github.com/Microsoft/mssql-docker/blob/master/linux/preview/examples/mssql-agent-fts-ha-tools/Dockerfile) in your directory (we are using this dockerfile because it has full text search installed)
### Environment Setup
* Follow instructions from this [website](https://schwabencode.com/blog/2019/10/27/MSSQL-Server-2017-Docker-Full-Text-Search) to create build the docker image and run the image (make sure to run this command in the same directory as the Dockerfile)
* Move the file to Docker by running this command (it might take few mins since the mdf file is pretty big): 
```
docker cp StackOverflow2010.mdf mssql:/var/opt/mssql/data/
docker cp StackOverflow2010_log.ldf mssql:/var/opt/mssql/data/
```
* Install mssql-tools, follow instructions on [Install tools on Ubuntu 20.04](https://docs.microsoft.com/en-us/sql/linux/sql-server-linux-setup-tools?view=sql-server-ver15): 
* Go to the SQL command
```
/opt/mssql-tools/bin/sqlcmd -S localhost -U sa -P [YourPassword]
```
* Create the Database with this sql code:
```sql
CREATE DATABASE Stackoverflow 
ON ( FILENAME = N'/var/opt/mssql/data/StackOverflow2010.mdf'), ( FILENAME = N'/var/opt/mssql/data/StackOverflow2010_log.ldf')
FOR ATTACH
GO
```
* Enable full text feature:
```sql
exec sp_fulltext_database 'enable'; 
```
* Create full text catalog:
```sql
CREATE FULLTEXT CATALOG FullTextCatalog AS DEFAULT;
```
* Create full text index on table Posts for column Body and Title
```sql
create fulltext index on Posts(Body, Title) key index PK_Posts__Id
```
* For more information on [Add Full Text Search on SQL Server](https://www.sqlshack.com/hands-full-text-search-sql-server/) and [What is full text search](https://docs.microsoft.com/en-us/sql/relational-databases/search/full-text-search?view=sql-server-ver15)


### Another problem that I encountered:
* Initially I made a rookie mistake to process SQL one element at a time, which resulted in very poor performance, looking up 50 rows would take about 10 seconds. However, I realized my mistake and wrote the sql query all together by joining the tables, now the search time is instant:
```sql
With rep AS (SELECT Reputation, Id as userID
FROM Users),

badge AS (SELECT Name, Id as uID
FROM Badges),

temp AS (SELECT TOP 100 Id, Title, Body, OwnerUserId, AnswerCount FROM Posts as temp 
INNER JOIN 
CONTAINSTABLE([Posts], (title, body), '"${req.params.query}"') as outcome 
On temp.Id = outcome.[KEY] 
ORDER BY outcome.RANK DESC),

rep_badge AS (select * from rep JOIN badge on rep.userID = badge.uID),

halfResult AS (select temp.Id, temp.Title, temp.Body, temp.OwnerUserId, temp.AnswerCount, COUNT(*) as totalVotes 
FROM temp LEFT JOIN Votes ON temp.Id = Votes.PostId 
GROUP BY temp.Id, Title, Body, OwnerUserId, AnswerCount)

SELECT halfResult.Id, halfResult.Title, halfResult.Body, halfResult.OwnerUserId, halfResult.AnswerCount, 
halfResult.totalVotes, rep_badge.Reputation, rep_badge.Name 
FROM halfResult 
LEFT JOIN rep_badge 
ON halfResult.OwnerUserId = rep_badge.userID`
```

### Question 2 (it has been tested already):
```SQL
WITH tmp AS 
  (SELECT DATENAME(WEEKDAY, CreationDate) AS Day_of_the_week, Id, PostTypeId 
  FROM Posts), 
postAndFeedback AS
  (SELECT PostTypeId, Day_of_the_week, f.Id, f.VoteTypeId 
  FROM tmp p 
  JOIN Votes f 
  ON p.Id = f.PostId), 
totalPost_q AS 
  (SELECT Day_of_the_week, COUNT(Id) AS Total_Posts 
  FROM tmp 
  WHERE PostTypeId = 1 
  GROUP BY Day_of_the_week), 
totalPost_a AS 
  (SELECT Day_of_the_week, COUNT(Id) AS Total_Posts 
  FROM tmp 
  WHERE PostTypeId = 2 GROUP BY Day_of_the_week), 
upVote_q AS 
  (SELECT Day_of_the_week, COUNT(*) AS Total_Upvotes 
  FROM postAndFeedback 
  WHERE VoteTypeId = 2 AND PostTypeId = 1 GROUP BY Day_of_the_week), 
upVote_a AS 
  (SELECT Day_of_the_week, COUNT(*) AS Total_Upvotes 
  FROM postAndFeedback WHERE VoteTypeId = 2 AND PostTypeId = 2 
  GROUP BY Day_of_the_week), 
downVote_q AS 
  (SELECT Day_of_the_week, COUNT(*) AS Total_downvotes 
  FROM postAndFeedback 
  WHERE VoteTypeId = 3 AND PostTypeId = 1 
  GROUP BY Day_of_the_week), 
downVote_a AS 
  (SELECT Day_of_the_week, COUNT(*) AS Total_downvotes 
  FROM postAndFeedback 
  WHERE VoteTypeId = 3 AND PostTypeId = 2 GROUP BY Day_of_the_week) 
SELECT 'Question' AS PostType, T1.Day_of_the_week, Total_Posts, Total_Upvotes, Total_downvotes, Total_Upvotes / Total_downvotes AS Upvotes_to_Downvotes_ratio 
FROM totalPost_q T1 
JOIN upVote_q T2 
ON T1.Day_of_the_week = T2.Day_of_the_week 
JOIN downVote_q T3 
ON T1.Day_of_the_week = T3.Day_of_the_week
UNION 
SELECT 'Answer' AS PostType, T1.Day_of_the_week, Total_Posts, Total_Upvotes, Total_downvotes, Total_Upvotes / Total_downvotes AS Upvotes_to_Downvotes_ratio 
FROM totalPost_a T1 
JOIN upVote_a T2 
ON T1.Day_of_the_week = T2.Day_of_the_week 
JOIN downVote_a T3 
ON T1.Day_of_the_week = T3.Day_of_the_week 
ORDER BY Upvotes_to_Downvotes_ratio DESC
```

### Question 3
* Since there is no obvious definition of an active user, so I decided that the user who created a post during the time frame would be an active user. 

```sql
WITH postDate AS
(SELECT CONVERT(varchar(100),(dateadd(DAY, -(datepart(weekday, CreationDate)+1), CreationDate)), 111) AS 
    first_date_of_the_week, Id, PostTypeId, OwnerUserId
FROM Posts),
    question AS
(SELECT first_date_of_the_week, COUNT(*) AS Count_of_questions
FROM postDate
WHERE PostTypeId = 1
GROUP BY first_date_of_the_week),
    answer AS
(SELECT first_date_of_the_week, COUNT(*) AS Count_of_answers
FROM postDate
WHERE PostTypeId = 2
GROUP BY first_date_of_the_week),
    accepted AS
(SELECT first_date_of_the_week, COUNT(*) AS Count_of_accepted_answers
FROM postDate
WHERE PostTypeId = 1 AND AcceptedAnswerId > 0
GROUP BY first_date_of_the_week),
    totalVotes AS
(SELECT first_date_of_the_week, COUNT(*) AS Count_of_votes
FROM postDate JOIN Votes ON Votes.PostId = postDate.Id
GROUP BY first_date_of_the_week),
    userDate AS
(SELECT CONVERT(varchar(100),(dateadd(DAY, -(datepart(weekday, CreationDate)+1), CreationDate)), 111) AS first_date_of_the_week, Id
FROM Users),
    newUsers AS
(SELECT first_date_of_the_week, COUNT(*) AS Count_of_new_users
FROM userDate
GROUP BY first_date_of_the_week),
    activeUsers AS
(SELECT first_date_of_the_week, COUNT(DISTINCT OwnerUserId) AS Count_of_active_users
FROM postDate
GROUP BY first_date_of_the_week)
SELECT T1.first_date_of_the_week,T1.Count_of_questions, T2.Count_of_answers, T3.Count_of_accepted_answers,  
    T4.Count_of_votes, T5.Count_of_new_users,T6.Count_of_active_users
FROM question T1 FULL OUTER JOIN answer T2 ON T2.first_date_of_the_week = T1.first_date_of_the_week
    FULL OUTER JOIN accepted T3 ON T3.first_date_of_the_week = T1.first_date_of_the_week
    FULL OUTER JOIN totalVotes T4 ON T4.first_date_of_the_week = T1.first_date_of_the_week
    FULL OUTER JOIN newUsers T5 ON T5.first_date_of_the_week = T1.first_date_of_the_week
    FULL OUTER JOIN activeUsers T6 ON T6.first_date_of_the_week = T1.first_date_of_the_week
ORDER BY first_date_of_the_week
```

