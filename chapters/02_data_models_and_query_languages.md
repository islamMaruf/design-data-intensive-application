# Chapter 2: Data Models and Query Languages

## Introduction

The data model is perhaps the most important decision you'll make when building an application. It profoundly affects not just how the software is written, but also how we think about the problem we're solving. Most applications are built by layering one data model on top of another:

1. **Application Layer**: Objects, data structures, APIs
2. **Database Layer**: Tables, documents, graphs, key-value pairs
3. **Storage Layer**: Bytes on disk, memory, SSDs
4. **Hardware Layer**: Electrical currents, magnetic fields, photons

Each layer hides the complexity of the layers below by providing a clean data model. These abstractions allow different groups of people to work together effectively.

In this chapter, we'll explore three main data models:
- **Relational Model** (SQL): Data organized in tables with rows and columns
- **Document Model** (NoSQL): Semi-structured documents with nested hierarchies
- **Graph Model**: Entities connected by relationships

We'll also examine various query languages and understand when each model shines.

## Part 1: Relational Model vs Document Model

### The Birth of Relational Databases

In 1970, Edgar F. Codd proposed the relational model. The main idea was simple: organize data into **relations** (tables), where each relation is a collection of **tuples** (rows).

**Key Benefits of Relational Model:**
- **Data Independence**: Hide implementation details behind a clean interface
- **Query Flexibility**: SQL allows complex queries without predefining access patterns
- **ACID Transactions**: Strong consistency guarantees
- **Normalization**: Reduce data duplication

**Example: Relational Schema**

```sql
-- Users table
CREATE TABLE users (
    user_id SERIAL PRIMARY KEY,
    username VARCHAR(50) UNIQUE NOT NULL,
    email VARCHAR(100) UNIQUE NOT NULL,
    created_at TIMESTAMP DEFAULT NOW()
);

-- Posts table
CREATE TABLE posts (
    post_id SERIAL PRIMARY KEY,
    user_id INT REFERENCES users(user_id),
    title VARCHAR(200),
    content TEXT,
    created_at TIMESTAMP DEFAULT NOW()
);

-- Comments table
CREATE TABLE comments (
    comment_id SERIAL PRIMARY KEY,
    post_id INT REFERENCES posts(post_id),
    user_id INT REFERENCES users(user_id),
    text TEXT,
    created_at TIMESTAMP DEFAULT NOW()
);
```

### The NoSQL Revolution

Around 2010, a new generation of databases emerged, collectively called "NoSQL" (Not Only SQL). The driving forces were:

1. **Scale**: Need for massive read/write throughput
2. **Flexibility**: Desire for dynamic schemas
3. **Specialization**: Some queries not well expressed in SQL
4. **Open Source**: Avoiding commercial database vendors

**Document Databases** (MongoDB, CouchDB) became popular for their flexibility:

```json
{
  "_id": "user123",
  "username": "sarah_dev",
  "email": "sarah@example.com",
  "profile": {
    "full_name": "Sarah Johnson",
    "bio": "Full-stack developer",
    "location": "San Francisco, CA"
  },
  "posts": [
    {
      "post_id": "post456",
      "title": "Getting Started with Microservices",
      "content": "Microservices are...",
      "created_at": "2024-01-15T10:30:00Z",
      "comments": [
        {
          "comment_id": "comment789",
          "user_id": "user999",
          "username": "john_smith",
          "text": "Great article!",
          "created_at": "2024-01-15T11:00:00Z"
        }
      ]
    }
  ],
  "created_at": "2023-06-01T08:00:00Z"
}
```

### Relational vs Document: The Trade-offs

**Document Model Advantages:**
1. **Schema Flexibility**: No need to predefine all fields
2. **Locality**: Related data stored together (better read performance)
3. **Natural Mapping**: Closer to application objects

**Relational Model Advantages:**
1. **Better Joins**: Efficient relationships between entities
2. **Many-to-One/Many-to-Many**: Better normalization
3. **No Data Duplication**: Single source of truth

**Real-World Example: LinkedIn Profile**

**Relational Approach (Normalized):**

```
┌─────────────────────────────────┐
│         users                   │
├─────────────────────────────────┤
│ user_id │ name    │ email       │
├─────────────────────────────────┤
│ 1       │ Sarah   │ s@email.com │
└─────────────────────────────────┘
         │
         │ (one-to-many)
         ▼
┌─────────────────────────────────────────┐
│         positions                       │
├─────────────────────────────────────────┤
│ position_id │ user_id │ company_id     │
│ start_date  │ end_date│ title          │
├─────────────────────────────────────────┤
│ 1           │ 1       │ 100            │
│ 2020-01     │ NULL    │ Senior Engineer│
└─────────────────────────────────────────┘
         │
         │ (many-to-one)
         ▼
┌─────────────────────────────┐
│         companies           │
├─────────────────────────────┤
│ company_id │ name           │
├─────────────────────────────┤
│ 100        │ Google         │
└─────────────────────────────┘
```

**Document Approach (Denormalized):**

```json
{
  "_id": 1,
  "name": "Sarah Johnson",
  "email": "sarah@email.com",
  "positions": [
    {
      "company": "Google",
      "title": "Senior Engineer",
      "start_date": "2020-01",
      "end_date": null,
      "description": "Leading infrastructure team"
    },
    {
      "company": "Microsoft",
      "title": "Software Engineer",
      "start_date": "2017-06",
      "end_date": "2019-12"
    }
  ],
  "education": [
    {
      "institution": "MIT",
      "degree": "MS Computer Science",
      "year": 2017
    }
  ]
}
```

### The Impedance Mismatch

**Problem**: Relational databases use tables and rows, but application code uses objects and data structures. This mismatch requires translation layers (ORMs).

**Example with ORM (Python SQLAlchemy):**

```python
from sqlalchemy import Column, Integer, String, ForeignKey
from sqlalchemy.orm import relationship

class User(Base):
    __tablename__ = 'users'
    
    id = Column(Integer, primary_key=True)
    username = Column(String(50), unique=True)
    email = Column(String(100), unique=True)
    
    # Relationship
    posts = relationship("Post", back_populates="user")

class Post(Base):
    __tablename__ = 'posts'
    
    id = Column(Integer, primary_key=True)
    user_id = Column(Integer, ForeignKey('users.id'))
    title = Column(String(200))
    content = Column(String)
    
    user = relationship("User", back_populates="posts")

# Usage
user = session.query(User).filter_by(username='sarah_dev').first()
for post in user.posts:
    print(post.title)
```

**Document databases** reduce this mismatch since JSON/BSON structure closely resembles objects.

### Many-to-One and Many-to-Many Relationships

**Many-to-One Example**: Many posts belong to one user
**Many-to-Many Example**: Many students enroll in many courses

**Relational Model Excels:**

```sql
-- Many-to-Many: Students and Courses
CREATE TABLE students (
    student_id SERIAL PRIMARY KEY,
    name VARCHAR(100)
);

CREATE TABLE courses (
    course_id SERIAL PRIMARY KEY,
    title VARCHAR(200)
);

-- Junction table
CREATE TABLE enrollments (
    student_id INT REFERENCES students(student_id),
    course_id INT REFERENCES courses(course_id),
    enrolled_at TIMESTAMP DEFAULT NOW(),
    PRIMARY KEY (student_id, course_id)
);

-- Query: Find all courses for a student
SELECT c.title, e.enrolled_at
FROM enrollments e
JOIN courses c ON e.course_id = c.course_id
WHERE e.student_id = 123;
```

**Document Model Challenge:**

```json
{
  "_id": "student123",
  "name": "Alice",
  "enrollments": [
    {"course_id": "course456", "title": "Databases 101"},
    {"course_id": "course789", "title": "Algorithms"}
  ]
}
```

**Problem**: If course title changes, we must update ALL student documents. This is data duplication.

**Solution**: Use references (like foreign keys):

```json
{
  "_id": "student123",
  "name": "Alice",
  "course_ids": ["course456", "course789"]
}
```

But now we need application-level joins!

## Part 2: Query Languages

### Declarative vs Imperative Queries

**Imperative**: Tells the computer HOW to do something (step by step)
**Declarative**: Tells the computer WHAT you want (optimizer decides how)

**Example Task**: Find all sharks from animals list

**Imperative Approach (JavaScript):**

```javascript
function getSharks(animals) {
  const sharks = [];
  for (let i = 0; i < animals.length; i++) {
    if (animals[i].family === 'Sharks') {
      sharks.push(animals[i]);
    }
  }
  return sharks;
}
```

**Declarative Approach (SQL):**

```sql
SELECT * FROM animals WHERE family = 'Sharks';
```

**Advantages of Declarative:**
- Database can optimize execution
- Can parallelize automatically
- More concise and readable
- Less error-prone

### SQL: The Relational Query Language

SQL (Structured Query Language) has been the dominant database language for 40+ years.

**Basic SQL Operations:**

```sql
-- SELECT: Read data
SELECT username, email FROM users WHERE created_at > '2024-01-01';

-- INSERT: Add data
INSERT INTO users (username, email) VALUES ('alice', 'alice@example.com');

-- UPDATE: Modify data
UPDATE users SET email = 'newemail@example.com' WHERE user_id = 123;

-- DELETE: Remove data
DELETE FROM posts WHERE created_at < '2020-01-01';
```

**Complex SQL Example: Analytics Query**

```sql
-- Find top 5 most active users (by post count) in the last month
SELECT 
    u.username,
    u.email,
    COUNT(p.post_id) as post_count,
    AVG(LENGTH(p.content)) as avg_post_length
FROM users u
LEFT JOIN posts p ON u.user_id = p.user_id
WHERE p.created_at >= NOW() - INTERVAL '30 days'
GROUP BY u.user_id, u.username, u.email
HAVING COUNT(p.post_id) > 5
ORDER BY post_count DESC
LIMIT 5;
```

**SQL Joins Explained:**

```
INNER JOIN: Only matching rows
┌──────────┐        ┌──────────┐
│  Users   │        │  Posts   │
├──────────┤        ├──────────┤
│ id | name│        │ id | uid │
├──────────┤        ├──────────┤
│ 1  | Alice│   ╱   │ 10 | 1   │
│ 2  | Bob  │ ⨯     │ 11 | 1   │
│ 3  | Carol│   ╲   │ 12 | 2   │
└──────────┘        └──────────┘

Result (INNER JOIN):
┌─────────────────────┐
│ name  │ post_id     │
├─────────────────────┤
│ Alice │ 10          │
│ Alice │ 11          │
│ Bob   │ 12          │
└─────────────────────┘

LEFT JOIN: All from left, matching from right
Result (LEFT JOIN):
┌─────────────────────┐
│ name  │ post_id     │
├─────────────────────┤
│ Alice │ 10          │
│ Alice │ 11          │
│ Bob   │ 12          │
│ Carol │ NULL        │
└─────────────────────┘
```

### NoSQL Query Languages

#### MongoDB Query Language

MongoDB uses JSON-like query syntax:

```javascript
// Find all users created in 2024
db.users.find({
  created_at: {
    $gte: ISODate("2024-01-01"),
    $lt: ISODate("2025-01-01")
  }
})

// Complex aggregation pipeline
db.posts.aggregate([
  // Stage 1: Match recent posts
  {
    $match: {
      created_at: { $gte: ISODate("2024-01-01") }
    }
  },
  // Stage 2: Group by user
  {
    $group: {
      _id: "$user_id",
      post_count: { $sum: 1 },
      total_length: { $sum: { $strLenCP: "$content" } }
    }
  },
  // Stage 3: Calculate average
  {
    $project: {
      user_id: "$_id",
      post_count: 1,
      avg_length: { $divide: ["$total_length", "$post_count"] }
    }
  },
  // Stage 4: Sort and limit
  {
    $sort: { post_count: -1 }
  },
  {
    $limit: 5
  }
])
```

**Aggregation Pipeline Visualization:**

```
Input Documents
       │
       ▼
   ┌────────┐
   │ $match │  Filter documents
   └────────┘
       │
       ▼
   ┌────────┐
   │ $group │  Group and aggregate
   └────────┘
       │
       ▼
   ┌─────────┐
   │ $project│  Transform fields
   └─────────┘
       │
       ▼
   ┌────────┐
   │ $sort  │  Order results
   └────────┘
       │
       ▼
   ┌────────┐
   │ $limit │  Limit output
   └────────┘
       │
       ▼
Output Documents
```

### MapReduce Querying

MapReduce is a programming model for processing large datasets across many machines. It was popularized by Google and implemented in MongoDB and Hadoop.

**MapReduce Concept:**

```
Input Data → Map → Shuffle & Sort → Reduce → Output

Example: Word Count
Input: ["hello world", "hello"]

Map Phase:
"hello world" → [("hello", 1), ("world", 1)]
"hello"       → [("hello", 1)]

Shuffle & Sort:
("hello", [1, 1])
("world", [1])

Reduce Phase:
("hello", [1, 1]) → ("hello", 2)
("world", [1])    → ("world", 1)

Output: {"hello": 2, "world": 1}
```

**MongoDB MapReduce Example:**

```javascript
// Count posts per user
db.posts.mapReduce(
  // Map function
  function() {
    emit(this.user_id, 1);
  },
  // Reduce function
  function(key, values) {
    return Array.sum(values);
  },
  {
    out: "post_counts"
  }
)
```

**MapReduce Limitations:**
- More verbose than SQL or aggregation pipelines
- Two-phase restriction (map then reduce)
- Performance overhead

**Modern Alternative: Aggregation Pipeline** (shown above) is preferred in MongoDB.

## Part 3: Graph-Like Data Models

Some applications have many-to-many relationships with complex connections. Examples:
- **Social Networks**: Who knows whom
- **Web Graph**: Which page links to which
- **Road Networks**: Route planning

### Property Graphs

**Model**: Vertices (nodes) and edges (relationships) with properties

**Example Schema:**

```
Vertices (Nodes):
┌──────────────────────────────┐
│ vertex_id │ label │ properties│
├──────────────────────────────┤
│ 1         │ Person│ {name:"Alice"}│
│ 2         │ Person│ {name:"Bob"}  │
│ 3         │ City  │ {name:"NYC"}  │
└──────────────────────────────┘

Edges (Relationships):
┌────────────────────────────────────────┐
│ edge_id │ tail │ head │ label │ props │
├────────────────────────────────────────┤
│ 10      │ 1    │ 2    │ KNOWS │ {}    │
│ 11      │ 1    │ 3    │ LIVES_IN│ {}  │
│ 12      │ 2    │ 3    │ LIVES_IN│ {}  │
└────────────────────────────────────────┘
```

**Visual Representation:**

```
    Alice (1)
      │  │
      │  └─────KNOWS────────┐
      │                     │
   LIVES_IN              LIVES_IN
      │                     │
      ▼                     ▼
   NYC (3) ◄─────────────  Bob (2)
```

**Relational Implementation:**

```sql
CREATE TABLE vertices (
    vertex_id SERIAL PRIMARY KEY,
    label VARCHAR(50),
    properties JSONB
);

CREATE TABLE edges (
    edge_id SERIAL PRIMARY KEY,
    tail_vertex INT REFERENCES vertices(vertex_id),
    head_vertex INT REFERENCES vertices(vertex_id),
    label VARCHAR(50),
    properties JSONB
);

-- Find all people Alice knows
SELECT v.*
FROM edges e
JOIN vertices v ON e.head_vertex = v.vertex_id
WHERE e.tail_vertex = (SELECT vertex_id FROM vertices WHERE properties->>'name' = 'Alice')
  AND e.label = 'KNOWS';
```

### Cypher Query Language (Neo4j)

Cypher is a declarative language for property graphs, designed to be intuitive.

**Basic Patterns:**

```cypher
// Create nodes and relationships
CREATE (alice:Person {name: 'Alice', age: 30})
CREATE (bob:Person {name: 'Bob', age: 25})
CREATE (nyc:City {name: 'New York'})
CREATE (alice)-[:KNOWS]->(bob)
CREATE (alice)-[:LIVES_IN]->(nyc)
CREATE (bob)-[:LIVES_IN]->(nyc)

// Find Alice's friends
MATCH (alice:Person {name: 'Alice'})-[:KNOWS]->(friend)
RETURN friend.name

// Find people who live in the same city as Alice
MATCH (alice:Person {name: 'Alice'})-[:LIVES_IN]->(city)<-[:LIVES_IN]-(person)
WHERE person <> alice
RETURN person.name, city.name

// Find all paths between Alice and a target
MATCH path = (alice:Person {name: 'Alice'})-[*..5]-(target:Person {name: 'Charlie'})
RETURN path
```

**Complex Example: Friend Recommendations**

```cypher
// Find friends of friends who Alice doesn't know
MATCH (alice:Person {name: 'Alice'})-[:KNOWS]->(friend)-[:KNOWS]->(fof)
WHERE NOT (alice)-[:KNOWS]->(fof) AND alice <> fof
RETURN fof.name, COUNT(*) as mutual_friends
ORDER BY mutual_friends DESC
LIMIT 10
```

### Triple-Stores and SPARQL

**Triple-Store Model**: Everything is a triple (subject, predicate, object)

**Example Triples:**

```
(Alice, knows, Bob)
(Alice, livesIn, NewYork)
(Alice, age, 30)
(NewYork, country, USA)
(USA, continent, NorthAmerica)
```

**RDF Format (Turtle):**

```turtle
@prefix : <http://example.com/> .

:Alice :knows :Bob ;
       :livesIn :NewYork ;
       :age 30 .

:NewYork :country :USA .

:USA :continent :NorthAmerica .
```

**SPARQL Query Language:**

```sparql
# Find all people Alice knows
SELECT ?friend
WHERE {
  :Alice :knows ?friend .
}

# Find people who live in North American cities
SELECT ?person ?city
WHERE {
  ?person :livesIn ?city .
  ?city :country ?country .
  ?country :continent :NorthAmerica .
}
```

### Graph Databases vs Relational with Joins

**Scenario**: Find all people within 3 degrees of separation from Alice.

**Relational Approach** (multiple self-joins):

```sql
-- This gets extremely complex for variable-length paths!
SELECT DISTINCT u3.*
FROM users u1
JOIN friendships f1 ON u1.user_id = f1.user_id
JOIN users u2 ON f1.friend_id = u2.user_id
JOIN friendships f2 ON u2.user_id = f2.user_id
JOIN users u3 ON f2.friend_id = u3.user_id
WHERE u1.name = 'Alice';
-- This only handles 2 degrees; 3+ requires recursive CTEs
```

**Graph Database Approach:**

```cypher
MATCH (alice:Person {name: 'Alice'})-[:KNOWS*1..3]-(person)
RETURN DISTINCT person.name
```

**Performance Difference:**
- **Relational**: Query time increases with join complexity
- **Graph**: Query time proportional to traversed portion (local index)

## Part 4: Schema Flexibility and Evolution

### Schema-on-Write vs Schema-on-Read

**Schema-on-Write** (Relational):
- Schema enforced when data is written
- Similar to static typing in programming
- Database rejects invalid data

```sql
CREATE TABLE users (
    user_id INT PRIMARY KEY,
    username VARCHAR(50) NOT NULL,
    age INT CHECK (age >= 0 AND age <= 150)
);

-- This will fail: age is required
INSERT INTO users (user_id, username) VALUES (1, 'alice');
```

**Schema-on-Read** (Document):
- Schema interpreted when data is read
- Similar to dynamic typing
- Application handles schema validation

```javascript
// MongoDB accepts any structure
db.users.insert({
  _id: 1,
  username: "alice"
  // age is optional
});

// Application handles missing fields
const user = db.users.findOne({_id: 1});
const age = user.age || "unknown";
```

### Schema Evolution Example

**Scenario**: Add a new field `phone_number` to users.

**Relational Approach:**

```sql
-- Add column (might lock table on large tables!)
ALTER TABLE users ADD COLUMN phone_number VARCHAR(20);

-- Set default for existing rows
UPDATE users SET phone_number = NULL WHERE phone_number IS NULL;
```

**Document Approach:**

```javascript
// Old documents
{_id: 1, username: "alice"}

// New documents
{_id: 2, username: "bob", phone_number: "+1-555-0123"}

// Application handles both
function displayUser(user) {
  const phone = user.phone_number || "Not provided";
  console.log(`${user.username}: ${phone}`);
}
```

**Hybrid Approach** (Best of both worlds):

```python
# Use database validation with flexibility
class User:
    def __init__(self, data):
        self.username = data['username']  # Required
        self.email = data.get('email', '')  # Optional
        self.phone = data.get('phone_number', None)  # Optional, new field
        
    def validate(self):
        if not self.username:
            raise ValueError("Username required")
        if self.email and '@' not in self.email:
            raise ValueError("Invalid email")
```

## Part 5: Data Locality and Performance

### Data Locality in Document Databases

**Advantage**: Related data stored together

```json
{
  "_id": "order123",
  "customer": {
    "name": "Alice",
    "email": "alice@example.com"
  },
  "items": [
    {"product": "Laptop", "price": 1200, "quantity": 1},
    {"product": "Mouse", "price": 25, "quantity": 2}
  ],
  "shipping_address": {
    "street": "123 Main St",
    "city": "New York",
    "zipcode": "10001"
  },
  "total": 1250,
  "created_at": "2024-01-15T10:00:00Z"
}
```

**Benefit**: One read operation fetches entire order

**Relational Equivalent** (requires multiple joins):

```sql
SELECT 
    o.order_id,
    c.name, c.email,
    oi.product, oi.price, oi.quantity,
    a.street, a.city, a.zipcode
FROM orders o
JOIN customers c ON o.customer_id = c.customer_id
JOIN order_items oi ON o.order_id = oi.order_id
JOIN addresses a ON o.shipping_address_id = a.address_id
WHERE o.order_id = 'order123';
```

**Disadvantage**: Updating embedded data can be expensive

```javascript
// To update customer email, entire document might need rewriting
db.orders.updateMany(
  {"customer.email": "old@example.com"},
  {$set: {"customer.email": "new@example.com"}}
);
```

### Column-Oriented Storage (Preview)

For analytics workloads, storing data by column rather than row is more efficient:

**Row-Oriented** (traditional):

```
Row 1: [user_id=1, name="Alice", age=30, city="NYC"]
Row 2: [user_id=2, name="Bob", age=25, city="LA"]
```

**Column-Oriented**:

```
user_id column: [1, 2, ...]
name column: ["Alice", "Bob", ...]
age column: [30, 25, ...]
city column: ["NYC", "LA", ...]
```

**Benefit**: Read only columns needed for query (covered in Chapter 3).

## Part 6: Convergence of Data Models

Modern databases are converging, borrowing features from each other:

**PostgreSQL** (relational) added:
- JSONB for document-like storage
- Full-text search
- Geographic data types (PostGIS)

```sql
-- JSON support in PostgreSQL
CREATE TABLE users (
    user_id SERIAL PRIMARY KEY,
    data JSONB
);

INSERT INTO users (data) VALUES 
('{"name": "Alice", "age": 30, "hobbies": ["reading", "hiking"]}');

-- Query JSON data
SELECT data->>'name' as name, data->>'age' as age
FROM users
WHERE data @> '{"hobbies": ["reading"]}';
```

**MongoDB** (document) added:
- Multi-document ACID transactions
- Schema validation
- SQL-like aggregation

```javascript
// Schema validation in MongoDB
db.createCollection("users", {
  validator: {
    $jsonSchema: {
      bsonType: "object",
      required: ["username", "email"],
      properties: {
        username: {
          bsonType: "string",
          description: "must be a string and is required"
        },
        email: {
          bsonType: "string",
          pattern: "^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\\.[a-zA-Z]{2,}$"
        },
        age: {
          bsonType: "int",
          minimum: 0,
          maximum: 150
        }
      }
    }
  }
});
```

## Real-World Case Studies

### Case Study 1: Instagram (Transition from PostgreSQL to Cassandra)

**Challenge**: Scale to billions of photos and users

**Solution**: 
- Kept PostgreSQL for user data (relational integrity)
- Moved feed data to Cassandra (write-heavy, denormalized)

```python
# PostgreSQL: User relationships
user = db.execute("SELECT * FROM users WHERE user_id = ?", (123,))

# Cassandra: User feed (denormalized for fast reads)
# Partition key: user_id, Clustering key: timestamp
feed = cassandra.execute("""
    SELECT * FROM user_feed 
    WHERE user_id = ? 
    ORDER BY posted_at DESC 
    LIMIT 50
""", (123,))
```

### Case Study 2: GitHub (Using MySQL + Elasticsearch + Git)

**Multi-Model Approach**:
- **MySQL**: User accounts, repositories metadata
- **Elasticsearch**: Code search
- **Git**: Version control (graph-like structure)

```
User Search → Elasticsearch
Repository Metadata → MySQL
Code History → Git (directed acyclic graph)
```

## Summary

**Choosing a Data Model:**

| Use Case | Best Model | Examples |
|----------|------------|----------|
| **Transactions, relationships** | Relational | Banking, e-commerce |
| **Hierarchical, evolving schema** | Document | Content management, catalogs |
| **Complex relationships** | Graph | Social networks, recommendations |
| **Time-series, metrics** | Columnar | Analytics, monitoring |
| **Key-value lookup** | KV Store | Session storage, caching |

**Query Language Trade-offs:**

| Language | Declarative | Expressive | Performance | Learning Curve |
|----------|-------------|------------|-------------|----------------|
| **SQL** | ✓ | High | Optimized | Medium |
| **Cypher** | ✓ | High (graphs) | Optimized | Medium |
| **MongoDB Query** | ✓ | Medium | Good | Low |
| **MapReduce** | ✗ | High | Variable | High |

**Key Takeaways:**
1. No single data model solves all problems
2. Relational model excels at relationships and consistency
3. Document model provides flexibility and locality
4. Graph model best for highly connected data
5. Modern apps often use multiple models (polyglot persistence)
6. Schema-on-read vs schema-on-write is a fundamental trade-off
7. Query language affects both productivity and performance

**Looking Ahead:**
In Chapter 3, we'll dive deeper into how databases physically store and retrieve data, exploring storage engines like B-trees and LSM-trees that underpin these data models.
