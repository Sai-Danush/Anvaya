# Knowledge Base Articles

## Entity Relationship Diagram [ERD]
It is a visual representation of the database design that shows:
1. Entities [Database Table]
2. Attributes [Fields/Columns in each table]
3. Relationships[How tables connect to each other]

### Types of Relationships
1. One-to-One (1:1): One record in Table A relates to exactly one record in Table B
2. One-to-Many (1:M): One record in Table A relates to many records in Table B
3. Many-to-Many (M:M): Multiple records in Table A relate to multiple records in Table B


## SQL Terminology

#### Primary Key 
A unique identifier for each and every record in the current database table.
#### Foreign Key
An identifier used to reference the primary key of a record in a different (“foreign”) table.
#### Indexes 
Indexes are special data structures that improve the speed of data retrieval operations on database tables. Think of them like the index at the back of a book - instead of reading through the entire book to find information on a specific topic, we can check the index to quickly find the right page number.

1. Structure: Indexes are typically implemented as B-trees (balanced trees), which maintain data in a sorted order and allow for logarithmic time searches, insertions, and deletions.
2. Storage: When you create an index, the database creates a separate data structure that contains:
    - The indexed column(s) values
    - Pointers to the actual rows in the table

Using Indexes allows for faster data retrieval, improved query performance but also results increased storage space requirements and slows down data modification operations.


