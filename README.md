# op-storage

An interface for interacting with a database as a document-based store. It supports various implementations for the underlying database, including relational and key-value databases.

### Goals

- **[Simplify code](#code)**

  It's often difficult to write code that interacts with a database - most frameworks are non-intuitive and offer light abstractions that still require esoteric platform-specific knowledge.

- **[Simplify schema management](#schema)**

  There are two general pieces to using a database in your application: managing its schema / indexes, and writing code to interact with objects. Your code is tightly coupled to a specific version of the schema / indexes, so I want to merge these pieces and declare my schema / indexes directly in my code.

- **[Simplify scaling](#scaling)**

  I want to make it impossible to write slow / non-indexed queries.

- **[Platform-independent](#platform-independent)**

  Most database frameworks are not truly platform-independent. This is important to facilitate testing, and it lets you switch to more scalable implementations when your needs justify the cost.


---


### Quick-start

``` python
import op_storage

db = op_storage.factory.build('in-memory')

# Initialize your schema
db.init('user', indexes=[ 'age' ])

# Interacting with specific objects
user_uuid = db.create('user', { 'name': 'Alex', 'age': 31 })
user = db.get('user', user_uuid)
print('User: %s' % user)
db.update('user', user_uuid, { 'name': 'Alex', 'age': 32 })
user = db.get('user', user_uuid)
print('User: %s' % user)
db.delete('user', user_uuid)
try:
  db.get('user', user_uuid)
except op_storage.exceptions.NotFoundException:
  print('User was deleted')

# Querying over multiple objects
db.create('user', { 'name': 'Alex', 'age': 31 })
db.create('user', { 'name': 'Kelly', 'age': 29 })
db.create('user', { 'name': 'Ben', 'age': 27 })
users = db.list('user', 20 <= db['age'], db['age'] < 30)
for user_uuid, user in users:
  print('User UUID: %s, User: %s' % (user_uuid, user))
```


---


## <a name="code"></a> Simplify code

The programming language you use tends to influence how you think about solving problems. It is cognitively difficult to jump between different languages, or even different styles / patterns within the same language. ORMs attempt to solve this problem, but their abstractions are often too light - you still end up using SQL concepts and writing what is effectively SQL code, thus requiring knowledge of the nuances of SQL. The op-storage interface attempts to solve these problems by allowing you to write index and query logic in simple python.

#### Write index logic as python functions

In SQL, you can index a particular field as-is, or you can apply a function to index it. Consider a 'user' object structured as follows `{ 'user': 'Alex', 'email': 'AlexExarhos@gmail.com', 'age': 31 }`. It's easy to add an index directly on the 'age' field. For the 'email' field, let's say we want to index by the lowercased value. In this case, SQL provides the somewhat esoteric 'lower' function, and we could create both indexes as follows:

``` sql
CREATE INDEX "user-age" ON users;
CREATE INDEX "user-lower_email" ON users ((lower(email)));
```

In the op-service interface, you can create complex indexes using python code:

``` python
db.init(
  'user',
  indexes=[
    Index('age', key=lambda data: data['age']), # The string 'age' is a shorthand for this
    Index('email_lower', key=lambda data: data['email'].lower())
  ]
)
```

There are a couple of caveats here:

- Your indexes cannot return null values
  
  Why? Firstly, we want to maintain pythonic principles and avoid esoteric, possibly platform-dependent, sort rules. For example, suppose the following list contains our indexed values:
  
  ``` python
  [ 31, 28, None, 27, None ]
  ```
  
  Attempting to sort this list in python will generate an error, as it doesn't make sense to compare null values to integers. We extend the same rational behavior to our index values.
  
  The second reason is more of an implementation detail. We provide a dynamically-typed interface, despite the fact that some implementations may by statically-typed (i.e. SQL). Internally, we infer the type based on the value returned by the index's 'key' function. As such, only types where we can infer the database mapping are allowed. You can see the allowed index types by running `db.SUPPORTED_INDEX_TYPES`.

  If you need to represent null values, you can typically find a workaround using sentinel values. For example, when dealing with only positive integers, you could use -1. For strings, use ''. For UUIDs, use UUID(int=0).

- Your indexes must be serializable
  
  This means that your 'key' functions must not depend on any un-scoped variables. Use of any in-scope imports is allowed, but strongly discouraged, as the version of those imports is determined outside the serialization context (i.e. by your "requirements.txt" file). The op-service interface supports serializing lambdas. As function serialization can be difficult to understand and the underlying mechanism is subject to change, we provide a function `test_index_fn` that you can apply to make sure your function behaves as expected:
  
  ``` python
  assert op_storage.test_index_fn(lambda data: data['email'].lower())({ 'email': 'ABC' }) == 'abc'
  ```

#### Write query conditions using python syntax

ORMs and other database adapters have implemented query functionality in different ways. Some outright allow you to embed pseudo-sql as a string. Few stick to native python constructs. For example, given a raw SQL query as follows:

``` sql
SELECT * FROM user WHERE age >= 30 AND age < 40 AND height_in = 73;
```

In MongoDB the same query is written as follows - note that we do need to know about a few esoteric keywords:

``` python
db.user.find({ 'age': { '$ge': 30, '$lt': 40 }, 'height_in': 73 })
```

In the op-storage interface, the same query is written as:

``` python
db.list('user', db['age'] >= 30, db['age'] < 40, db['height_in'] == 73)
```

The bracket syntax is simply to reduce verbosity - it is really an alias for the underlying and more clearly-named `create_condition` function. But the point remains that these queries are much easier to read / write as they're built using regular python operators. There are many restrictions placed upon these conditions, most importantly they cannot be used with boolean expressions, and as a result cannot be chained.

Note that similarly to AWS DynamoDB, you can only apply all conditions at once (effectively an "and" operation). You could mimic an "or" operator by combining multiple queries (but a better solution is probably to build better indexes):

``` python
db.list('user', db['age'] == 31) + db.list('user', db['age'] == 32)
```

#### Auto-generate IDs 

SQL does not necessarily enforce that a table contains an indexed ID field. Traditionally, an ID field is explicitly created using an auto-incrementing integer, which is generally considered bad practice as it can reveal the size of your data set. The best practice is to use something like a UUID as the primary key. NoSQL databases such as MongoDB typically do a good job of automatically attaching an ID to every object, although sometimes the implementation can get murky. For example, MongoDB explicitly injects an '_id' property into your object, which could lead to conflicts.

The op-storage interface automatically generates UUIDs for every object and stores them separately without ever attaching additional ghost properties to objects:

``` python
my_uuid = db.create('user', { 'name': 'Alex', 'email': 'AlexExarhos@gmail.com', 'age': 31 })
db.get('user', my_uuid) -> { 'name': 'Alex', 'email': 'AlexExarhos@gmail.com', 'age': 31 }
db.list('user') -> [ (<uuid>, { 'name': 'Alex', 'email': 'AlexExarhos@gmail.com', 'age': 31 }) ]
```


---


## <a name="schema"></a> Simplify schema management

The "schema" of free-form document-based databases effectively only consists of data types and indexes on those data types. No structure is necessarily imposed on the documents themselves. Traditionally, a database schema is defined by modifications over time as requirements and design change, often described as "migrations". It is much clearer and simpler to specify the desired schema in your code directly.

For example, consider the following timeline (each item might be a separate deployment of production code):

1. Create an initial 'user' data type shaped like { 'name': str, 'email': str, 'age': int }

2. We add a new function that queries by 'age', so add an appropriate index over that field

3. We need to add an additional property 'height_in' and a corresponding index

4. We remove the function that queries for 'age', so no need for that index any more

In a SQL database, those steps might be accomplished as follows (potentially simplified by an ORM, but not fundamentally changed):

1. ``` sql
   CREATE TABLE user (id serial, name text, email text, age int, PRIMARY KEY(id));
   ```

2. ``` sql
   CREATE INDEX "user-age" ON user(age);
   ```

3. ``` sql
   ALTER TABLE user ADD COLUMN height_in int; CREATE INDEX "user-height_in" ON user(height_in);
   ```

4. ``` sql
   DROP INDEX "user-age";
   ```

Document-based / NoSQL databases simplify this process to some extent, as we can skip the field definition steps, but they don't fully remove the concept of migrations. These steps show the process using MongoDB:

1. ``` python
   db.createCollection('user') # Can be created on-the-fly, but it must exist before we add an index
   ```

2. ``` python
   db.user.createIndex({ 'age': 1 }) # '1' is an esoteric value meaning 'ascending'
   ```

3. ``` python
   db.user.createIndex({ 'height_in': 1 }) # No need to specify the new field for a free-form document-based database
   ```

4. ``` python
   db.user.dropIndex('age')
   ```

Note that with the previous example for MongoDB, it might be tempting to simply run all of those commands (as migrations) each time we run our code that needs to interract with the database, but we must be cognizant of the fact that there might be other versions of code running that depend on a different database state. For example, the call to `db.user.dropIndex('age')` will affect the queries for running code that still depend on that index (potentially breaking our application). Typically, we'd address this by waiting some arbitrary amount of time for all versions of code relying on that index to stop running, and only then would we remove it. This situation may sound obscure, but it is very common during rolling deployments or production testing - you will almost certainly have multiple versions of code running at some point.

The op-storage interface addresses this problem by allowing you to declare the current state of your "schema" (data types and indexes) instead of providing migration steps. It handles all the "migration" logic for you, including auto-removing indexes that have not been used in a while (running an old schema that specifies those indexes will still work, they will just have to be rebuilt). Importantly, multiple versions of this schema can run simultaneously wihout interfering with each other.

Using the op-storage interface, our timeline steps translate as follows - note that importantly, they are no longer "ordered steps" but rather "states" that we can attach to different versions of our code:

1. ``` python
   db.init('user')
   ```

2. ``` python
   db.init('user', indexes=[ 'age' ])`
   ```

3. ``` python
   db.init('user', indexes=[ 'age', 'height_in' ])
   ```

4. ``` python
   db.init('user', indexes=[ 'height_in' ])
   ```

Importantly, all these versions of code are compatible and can be run at the same time without interfering with each other. See the `test_index_lifecycle` test case for a more thorough example of how this works in a production context.


---


## <a name="scaling"></a> Simplify scaling

The op-service interface helps prevent scaling issues by disallowing non-indexed queries. Most databases make it possible to query data in a "slow" way (i.e. by performing a full table scan). Consider the following in SQL:

``` sql
CREATE TABLE user (id serial, name text, email text, age int, PRIMARY KEY(id));
SELECT * FROM user WHERE name = 'Alex';
```

Suppose the latter statement is wrapped up into a web API that is expected to be performant. During development and testing, that query will work, and that code may even run in production without any issue for some time. However, as we have no index on the 'name' field in the 'user' table, the execution time of that query is O(n) - proportional to the size of the 'user' table, and our web API will take so long as to effectively break when we grow to large numbers of users. This scenario is somewhat common in the real world - developers don't necessarily understand the nuances of SQL, and it's easy to miss these cases in testing as the code works for small data sets. Optimizing SQL queries / indexes can be a full-time job.

MongoDB has a similar shortcoming - it's possible to store and query data without indexes:

``` python
db.createCollection('user')
db.user.find({ 'name': 'Alex' })
```

AWS DynamoDB does a good job of logically separating the concept of a "query" from a full table scan, so it does not have this problem. You can't write a slow query in AWS DynamoDB. The same is true for the op-service interface.

In the op-storage interface, when you build conditions for your queries, you can only reference indexes that you've already created, so your queries are guaranteed to be performant. The following code will fail since 'name' is not a valid index on 'user':

``` python
db.init('user')
db.list('user', db['name'] == 'Alex') # Exception
```

If you want to query on 'name', you must initialize the 'user' data type with a 'name' index:

``` python
db.init('user', indexes=[ 'name' ])
db.list('user', db['name'] == 'Alex')
```


---


## <a name="platform-independent"></a> Platform-independent

The op-storage interface is strict and limited, which means it's easy to change the backend database implementation easily. Most database frameworks are not truly platform-independent in that you may need to write slightly different code for different implementations. Platform-independence is important for a couple of reasons:

- To facilitate testing
  It is common to spin up a clean database instance for testing. Implementations like the 'in-memory' store make this effortless.

- To facilitate scaling
  More scalable databases tend to be more expensive. Having a strict interface means that you can swap out your underlying database implementation without making any code changes.


---


### Shortcomings

- The op-storage interface does not necessarily minimize storage space. A tightly controlled SQL table with very specific types will take up less space.

- Data is not necessarily exposed for access outside of the op-service interface. For example, we impose no format restrictions on data type names. The following code is perfectly valid:

  ``` python
  db.init('Errors / Issues / Alarms', indexes=[ 'created_at', 'dedupe_key' ])
  ```
  
  But using a Postgres implementation, our corresponding table might look like this, which is pretty unintelligible to anyone trying to query the database directly:
  
  ```
  TABLE: op_69rrors32473273ssues32473265larms
  id     uuid          NOT NULL PRIMARY KEY
  data   bytea
  i_1    timestamptz   NOT NULL
  i_3    text          NOT NULL
  ```
  
  This point strays into principles of service design - a database should be tightly coupled to only one client. At many companies, it's common to provide database access to other teams (possibly non-technical, i.e. a BI team) so they can do their own analysis of the data. This almost invariably leads to problems as developers often use somewhat esoteric and non-intuitive formats / conventions to name fields that will lead to confusion and incorrect analyses. For example, let's say we're building a marketplace where users can buy and sell products from each other. Say a developer adds a 'num_purchases' field to our 'user' object. That developer might intend the field to mean "number of times other users have purchased from this user" and store appropriate data. A BI team might see that field and infer it to mean "number of purchases this user has made" and build their analyses around that misconception. Obviously more communication / documentation can help solve this problem, but a better solution is to force the developer to write a specific interface that exposes the exact data the BI team needs in a well-defined format.

- Initializing a schema using `db.init(...)` can be very slow if it has to create new index values for a large table. This code blocks and prevents use of an uninitialized index, but it might lead to confusion as to why a service is taking so long to deploy after a schema change. This is just a paradigm shift - in a traditional SQL database, we think of adding indexes as a slow external process. In fact, you still can run that initialization code externally, though that defeats some of the advantages discussed in the section on "state management".

- Complexity - there are a lot of moving pieces behind the scenes in the op-storage implementations. For example, an SQL implementation might start some background threads that periodically report indexes in use. More complexity means that there's more risk of failure - in the aforementioned case, we might have an envionrment where creating regular python threads is not supported. This problem will be mitigated as the framework becomes more mature.


---


# Development


Run the tests - note that some tests over specific implementations require a config to be specified:

``` bash
python3 setup.py test
```

Build and install the package locally:

``` bash
pip3 uninstall op-storage -y
pip3 install .
```

Build the package for distribution:

``` bash
python3 setup.py clean --all && python3 setup.py build && python3 setup.py sdist
```
