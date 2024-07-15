Relational database gotchas

If you use a relational database like MySQL or PostgreSQL to store data, there are
more than a few potential pitfalls worth paying attention to.
Taming your queries

It goes without saying that inefficient queries will give you inefficient scaling, slowing
down as the amount of data increases and the number of requests increase. To keep
things under control, I recommend logging and analyzing your queries and starting
early in the development process (you don’t want to wait until your app is a hit to look
at the queries!).
You can’t improve what you don’t measure, so logging the performance of SQL queries that are performed during each request is the most important first step. Look for
requests that generate a lot of queries, or slow queries, and start there.
Both MYSQL and PostgreSQL support the EXPLAIN command, which can help analyze
specific queries for performance. Common tactics to improve performance include
adding indices for commonly searched columns and reducing the number of JOINs
you need to perform. MySQL’s documentation “Optimizing SELECT Statements”a
goes into great detail on many different optimization tactics.


Avoiding N+1 queries


Even if your queries are superefficient, each individual query you make to the database has overhead. Ideally, each request your application processes should perform
a constant number of queries, regardless of how much data is displayed.
If you have a request that renders a list of objects, you ideally want to serve this
request without generating a separate query for each of those objects. This is commonly referred to as the N+1 query problem (as when the problem occurs, there is
often one query to get the list and then one for each item [N items] in the list).
This antipattern is particularly common with systems that use object-rational mapping
(ORM) and feature lazy loading between parent and child objects. Rendering the child
objects of a one-to-many relationship with lazy loading typically results in N+1 queries
(one query for the parent and N queries for the N child objects), which will show up
in your logs. Fortunately, there is normally a way with such systems to indicate up
front that you plan to access the child objects so that the queries can be batched.
Such N+1 query situations can normally be optimized into a constant number of queries, either with a JOIN to return the child objects in the list query or two queries: one
to get the record set and a second to get the details for the child objects in that set.
Remember, the goal is to have a small constant number of queries per request, and
in particular, the number of queries shouldn’t scale linearly with the number of
records being presented.

Using read replicas for SELECT queries

One of the best ways to reduce the strain on your primary database is to create a
read replica. In cloud environments, this is often really trivial to set up. Send all your
read queries to your read replica (or replicas!) to keep the load off the primary
read/write instance.
To design your application with this pattern in mind before you actually need a read
replica, you could have two database connections in your application to the same
database, using the second to simulate the read replica. Set up the read-only connection with its own user that only has read permissions. Later, when you need to
deploy an actual read replica, you can simply update the instance address of your
second connection, and you’re good to go!

Incrementing primary keys

If you really hit it big, you may end up regretting using incrementing primary keys.
They’re a problem for scaling, as they assume a single writable database instance
(inhibiting horizontal scaling) and require a lock when inserting, which affects performance (i.e., you can’t insert two records at once).
This is really only a problem at very large scale, but worth keeping in mind as it’s harder
to rearchitect things when you suddenly need to scale up. The common solution to this
problem is global UUIDs (e.g., 8fe05a6e-e65d-11ea-b0da-00155d51dc33), a 128-bit
number commonly displayed as a hexadecimal string, which can be uniquely generated
by any client (including code running on the user’s device).
When Twitter needed to scale up, it opted, instead, to create its own global incrementing IDs to retain the property that they are sortable (i.e., newer tweets have a
higher ID number), which you can read about in their post “Announcing Snowflake.”b
On the other hand, you might prefer to keep incrementing primary keys for aesthetic
reasons, like when the record ID is exposed to the user (as in the case of a tweet ID)
or for simplicity. Even if you plan to keep your incrementing primary keys for a while,
one step you can still take early on is not using auto-incrementing primary keys in places where they wouldn’t add any value, like, say, a user session object—maybe
not every table needs an incrementing primary key.