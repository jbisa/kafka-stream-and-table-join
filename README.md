# Join Kafka Stream and Table

There are times you may need to some filtering or data aggregation between several Kafka topics. One way to
achieve this, is by joining a Kafka `stream` with a Kafka `table`.

A **stream** is a set of continuous, immutable data. It is an append-only concept, where existing events (aka
records/messages) cannot be modified. Streams are persistent, durable, and fault tolerant. Events in a stream
can be keyed, and there is a one-to-many mapping between keys and events. A stream is basically like a table
in a relational DB (RDBMS), that has no unique key constraint and that is append-only.

A **table** on the other hand, is a set of immutable data. New events (or rows/records/messages) can be inserted,
updated, and deleted. Like streams, tables are persistent, durable, and fault tolerant. A table is basically a
materialized view in RDBMS, because it is being changed automatically under-the-hood, as soon as any of its
input streams or tables change (you don't actually need to manually insert, update, or delete records).

Although streams and tables are meant to be used to solve different problems, they are very similar concepts.
Data from a stream can be easily turned into a table and vice versa. This is known as the stream-table
duality.

# Dependencies

- Docker

# How to Run This Project

## Run Locally Using ksqlDB CLI

First, go to the project root directory and run the following, to set up your docker environment:

```docker-compose up -d```

Start the ksqlDB CLI locally, so that we can run some queries:

```docker exec -it ksqldb-cli ksql http://ksqldb-server:8088```

### Create the Necessary Tables

Create the `movies` table which will contain movie data:

```
CREATE TABLE movies (ID INT PRIMARY KEY, title VARCHAR, release_year INT)
    WITH (kafka_topic='movies', partitions=1, value_format='avro');
```

Insert data into the `movies` table:

```
INSERT INTO movies (id, title, release_year) VALUES (294, 'Die Hard', 1998);
INSERT INTO movies (id, title, release_year) VALUES (354, 'Tree of Life', 2011);
INSERT INTO movies (id, title, release_year) VALUES (782, 'A Walk in the Clouds', 1995);
INSERT INTO movies (id, title, release_year) VALUES (128, 'The Big Lebowski', 1998);
INSERT INTO movies (id, title, release_year) VALUES (780, 'Super Mario Bros.', 1993);
```

### Create the Necessary Streams

The `ratings` stream will contain events when users submit ratings for various movies:

```
CREATE STREAM ratings (MOVIE_ID INT KEY, rating DOUBLE)
    WITH (kafka_topic='ratings', partitions=1, value_format='avro');
```

Insert data into the `ratings` stream:

```
INSERT INTO ratings (movie_id, rating) VALUES (294, 8.2);
INSERT INTO ratings (movie_id, rating) VALUES (294, 8.5);
INSERT INTO ratings (movie_id, rating) VALUES (354, 9.9);
INSERT INTO ratings (movie_id, rating) VALUES (354, 9.7);
INSERT INTO ratings (movie_id, rating) VALUES (782, 7.8);
INSERT INTO ratings (movie_id, rating) VALUES (782, 7.7);
INSERT INTO ratings (movie_id, rating) VALUES (128, 8.7);
INSERT INTO ratings (movie_id, rating) VALUES (128, 8.4);
INSERT INTO ratings (movie_id, rating) VALUES (780, 2.1);
```

Configure the `ratings` stream, such that you consume from the very beginning of the topic:

```SET 'auto.offset.reset' = 'earliest';```

Create the `rated_movies` stream, which will be a join of the movies table and ratings stream:

```
CREATE STREAM rated_movies
    WITH (kafka_topic='rated_movies',
          value_format='avro') AS
    SELECT ratings.movie_id AS id, movies.title, movies.release_year, ratings.rating
    FROM ratings
    LEFT JOIN movies ON ratings.movie_id = movies.id;
```

### Check Results

To see what's in the `rated_movies` stream, run the following command:

```PRINT rated_movies FROM BEGINNING LIMIT 9;```

## Run Project Tests

Run the following command to run the tests locally:

```
docker exec ksqldb-cli ksql-test-runner -i /opt/app/test/testData.json -s /opt/app/src/statements.sql -o /opt/app/test/testResults.json
````
