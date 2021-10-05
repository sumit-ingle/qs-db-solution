# DB solution

## Schema
```sql
post(id: BIGINT, title: VARCHAR(100), description: VARCHAR(1000), created_date: DATETIME)
     1           example_title        sample_description          2021-10-04T00:00:00

comment(id: BIGINT, text: VARCHAR(200), postId: BIGINT FK post(id))   
        1           comment1              1   
        2           comment1.1            1   
        3           comment1.2            1   
        4           comment1.1.1          1   

comment_mapping(id: BIGINT, child_comment_id: BIGINT)   
                1            2   
                1            3   
                2            4   
```
```
post 1 <-> N comment
comment 1 <-> N comment
```
**Note**: I also realized that we don't need comment_mapping table. We can just use parent_comment_id in the comment table. And if we're building something like reddit, probably don't need the comment table either! We can have the `post` table like this:  
```
id   |   title    |   text     |   created_date     |   parent_post_id
-----------------------------------------------------------------------
1    |   title1   |   text1    |   created_date1    |   null
2    |   title2   |   text2    |   created_date2    |   1
3    |   title3   |   text3    |   created_date3    |   1

```

**But for below solutions, I've still used the older schema.**

## Q1. Find root comment id for comment1.1.1 (id = 4)
```sql
CREATE TABLE comment_mapping
(
    id int,
    child_comment_id int
);

INSERT INTO comment_mapping (id, child_comment_id) VALUES(1, 2);
INSERT INTO comment_mapping (id, child_comment_id) VALUES(1, 3);
INSERT INTO comment_mapping (id, child_comment_id) VALUES(2, 4);

;WITH cte_comment AS (
  -- step 1: we insert required child row into the cte, with initial row level = 1
  SELECT id, 1 AS [level]
  FROM comment_mapping
  WHERE child_comment_id = 4

  UNION ALL

  -- step 2: we recursively find the parent rows & increase the level count for each parent
  SELECT cm.id, child.level + 1
  FROM comment_mapping cm, cte_comment child
  WHERE cm.child_comment_id = child.id
)

-- step 3: finally, we fetch the id which has highest level (i.e the root will have highest level) 
SELECT TOP 1 id FROM cte_comment ORDER BY [level] desc;
```
`Output: 1`

## Q2. Write SQL query required for *Get posts* API above

### Approach 1:
Given pageSize = 10 (pre-determined or can take it from the user) & we receive page_number as input in the query:

#### Implementation for SQL Server:
```sql
-- query begins here
DECLARE @page_size INT
DECLARE @page_number INT
DECLARE @offset INT

SET @page_size = 10
SET @page_number = 2
SET @offset = @page_size * (@page_number - 1)

-- example: 
-- if @page_number = 1, @offset = 10 * (1 - 1) = 0, here we'll fetch 10 records between 1 to 10
-- if @page_number = 2, @offset = 10 * (2 - 1) = 10, here we'll fetch 10 records between 11 to 20
-- if @page_number = 3, @offset = 10 * (3 - 1) = 20, here we'll fetch 10 records between 21 to 30

SELECT *  FROM post p 
INNER JOIN comment c ON p.id = c.postId
WHERE c.id NOT IN (SELECT child_comment_id FROM comment_mapping) -- 1st level comment will not be in child_comment_id column
ORDER BY date desc
OFFSET @offset ROWS
FETCH NEXT @page_size ROWS ONLY
-- query ends here
```

#### Same logic can be written in PostgreSQL like this:
```sql
SELECT *  FROM post p 
INNER JOIN comment c ON p.id = c.postId
WHERE c.id NOT IN (SELECT child_comment_id FROM comment_mapping) -- 1st level comment will not be in child_comment_id column
ORDER BY date desc
LIMIT @page_size
OFFSET @offset
```

### Approach 2:
Given pageSize = 10 (pre-determined) & we receive offset_date as an input in the query.  
Here we retrieve 10 posts starting from offset date.  
Initially, when user is on 1st page, offset date will todays date time. So, we'll return 10 most recent posts. 
When user wants to navigate to the 2nd page, we'll send the date of last post on the first page as input. So for 2nd page, we'll return 10 posts that come after that date.

#### Implementation (SQL Server):
```sql
-- query begins here
DECLARE @page_size INT
DECLARE @offset INT

SET @page_size = 10
SET @date_offset = '2021-10-05T00:00:00'

-- example: 
-- if @page_size = 10, @date_offset = CURRENT_TIME, here we'll fetch 10 most recent records
-- if @page_size = 10, @date_offset = '2021-10-04T00:00:00', here we'll fetch 10 records after 6th october 12 AM
-- if @page_size = 10, @date_offset = '2021-10-03T00:00:00', here we'll fetch 10 records after 3rd october 12 AM

SELECT *  FROM post p 
INNER JOIN comment c ON p.id = c.postId
WHERE c.id NOT IN (SELECT child_comment_id FROM comment_mapping) -- 1st level comment will not be in child_comment_id column
AND p.created_date <= @date_offset
ORDER BY date desc
LIMIT @page_size
-- query ends here
```
