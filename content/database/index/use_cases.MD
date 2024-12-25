# Analyze some use cases

I'll show you some real-life case that if you use index the right way, it'll reduce so much time in querying

## Delete a table with foreign key

I have a table named `survey_locations` contains locations that need to be surveyed. This table reference to two other tables: `survey_location_statistics` and `user_posts`

`survey_location_statistics` has `survey_location_id` as its Primary Key.
`user_posts` has `survey_location_id` in it, just a normal column, no index.

So, my task is to remove data in `survey_locations`. It has about 500000 rows, and need to remove about 400000 (80% of the rows), it's moderate, not large so I expected it will run kinda fast.

But it takes forever to run. Why? (You can take sometime to guess where the problem lies)

I ran a `explain analyze ...`, and this is the result:

![Query result](https://depresseddeveloper.cloud/media/images/sharing/indx_usecases_delete_data.png)

You can see, I only remove 100 rows but takes about 6s to complete it. TOO SLOW.

Here is what I analyze:
- Query planner will use seq scan on table `survey_locations` to filter rows need to be removed (Because the number of deleting rows is too large, 80%, so it)
- It takes 0.249 to get the data to remove it
- And then, it has to validate reference of these 100 records before deleting it, and this is where the problem is.
  - Time to check constraints in table survey_location_statistics only 6.304 ms (call 100 times) -> kind of fast because the survey_location_id in survey_location_statistics is Primary Key, it means that this table indexed by this column
  - Time to check constraints in table user_posts takes 5939.886 ms (~6s) (also call 100 times) -> This is the slow part. Because we do not index survey_location_id in user_posts, so when validating constraints, it will scan full table user_posts.

You can see, the reason this query soo slows is that we do not index the column `survey_location_id` in `user_posts`.

Why I didn't index it in the first place? Because there is NO select query that needs to filter by that field, so I decided to not index that field.

*So*, you need to index the field not only for selecting, but also for deleting, updating, inserting.

Another approach could be removing foreign key, but it comes with a cons is the data integrity may not be maintained.

## Queries don't use index 

You can see that the query planner does not always use index. It only uses index in case the number of scanned rows is low enough. But it doesn't mean we let that query scan full table, right?
We need to find another constraint to put to the query, so the query may be a little bit more complex, but it'll reduce so much time to get the result. (More complex, less time, huh?)

Here is an example.

I have a table `locations`, store latitude, longitude and location name. It has about 3 million rows. These are locations for users to find to do the surveys. User has to get to the location close enough so they can take survey at that location (assume `n` meters).
This table has a composite index on (latitude, longitude)

There is a table named `posts` saves user's surveyed (include survey locations).

My problem is, I has user's location, I need to get the nearest location name to fill in tho this table, based on survey locations. Clear enough, right?

Take some times to think the solution and compare with mine.

So, the basic idea is we scan all latitude and longitude in `locations` table, get the location name and put it in `user_posts`, right?

```sql
select location_name
from locations
order by earth_distance(ll_to_earth({survey_latitude}, {survey_longitude}), ll_to_earth(latitude, longitude)) asc
limit 1
```

This query take about 6s to complete (The function `earth_distance` to get the distance between two coordinates in meter)

So you can see, query planner definitely doesn't use index, because it still scan full table event using index.
If we can reduce the number of scanned rows, we can also reduce the number of times use `earth_distance` function (which is quite expensive)

My solution is find another trait to query.
We have a condition when user doing the survey: User has to get to the location close enough so they can take survey at that location (assume `n` meters).

So what? How can we apply this condition into the query?

We can filter essential locations first by calculating the latitude max, latitude min, longitude max, longitude min from survey location (Because we has the maximum distance between survey location and the location).

![Nearest location](https://depresseddeveloper.cloud/media/images/sharing/idx_nearest_locations.png)

So, the query will be:

```sql
select location_name
from locations
where
    latitude between {latitude_min} and {latitude_max}
    and longitude between {longitude_min} and {longitude_max}
order by earth_distance(ll_to_earth({survey_latitude}, {survey_longitude}), ll_to_earth(latitude, longitude)) asc
limit 1
```

The above query reduced query time to 0.425 ms (very much faster than the old one!)
