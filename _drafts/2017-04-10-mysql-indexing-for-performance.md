---
layout: post
title: "MySQL Indexing for Performance"
date: 2016-09-10
categories: MySQL
tags: MySQL
comments: true
analytics: true
---

B-Tree Index
  - Most used index
  - Values stored in order
  - Each leaf page is at the same distance from root level.
  - Stores keys and data in internal leaf nodes.

InnoDB uses B+Tree Index
  - More indexes can be stored in memory
  - Intermediate node does not contain data therefore more keys can fit into the pages.
  - Data is only stored in leaf nodes
      - Internal nodes only contain pointers
  - Size of B+tree is much smaller than B-tree


Use a B-tree because no matter the size of the B-tree, traversal always takes the same amount of time.
Root -> Intermediate -> Leaf


Advantages of B-Tree Index
  - B-tree index speeds up data access
      - Storage engine traverses from root node to leaf node with the help of pointers.
  - Increased performance of following query patterns:
      - Full value ("Kyle Schmidt", "Kepler Group")
      - Leftmost Value or Column Prefix ("Kyle" from "Kyle Schmidt")
      - Range of Values (1 to 99, Aaron to Fritz)
      - Helps ORDER BY clause to increase performance.


Cluster Index
  - Different approach of data storage
      - Not really different type of index
      - Not all storage engines support it
  - Rows with adjacent key values are stored close to each other.
  - One clustered index per table.


Advantages of Cluster Index
  - Related data is stored close to each other leading to less disk I/O while retrieving sequential or range data.
      - Rather than doing a full table scan for indexes that are similar.
  - Faster data access as data and index are stored together at leaf nodes.
      - Leaf nodes point to each other


Disadvantages of Cluster Index
  - Data insertion speed is dependent on the order of the primary key
      - Table needs to be optimized if inserted data is not ordered by primary key.
  - Cluster index have minimal impact for in memory data.
  - Updating the cluster index column is expensive as data is moved based on its size to different locations.
  - Page split occurs when new data is inserted leading to fragmentation.
      - Leads to poor performance.
  - Secondary Index contains the location of the clustered index instead of row pointer, hence the size is larger.


Seconday Index
  - Index which is not clustered index is called a secondary index.
  - In InnoDB does not store actual data but only contains the pointer to the data.
  - If there is a cluster index on the table
      - Secondary Index will contain the pointer to the clustered index.
  - If there is no cluster index on the table
      - Secondary Index will contain the row pointer to the data.
  - We need Secondary Index because it is not possible to optimize all of our queries.
      - First query could be on first name
      - Second query could be on last name
          - Need two different indexes
              - One to order first _or_ last name
              - Second to order by the other
      - Last name as second index can _point_ to cluster index of first name.
      - Last name only query will require the secondary key index to find the appropriate last name and the pointer will jump back to the clustered primary key index.


Hash Index
  - Built on hash table
  - Increases performance for _exact_ lookups
  - For each row a hash code is generated
      - Generally different keys generate different hash codes
      - Stores hash codes in index with pointer to each row in a hash table.
      - If multiple values have same hash code, it will also contain their row pointers in linked list.
  - Memory storage engine in MySQL supports explicit hash tables
  - Very fast and effective as it resides in _memory_


Limitations of Hash Index
  - Only contains hash code and pointers and _not_ real data.
  - A hash index does not contain original data and is not effective in:
      - Sorting
      - No partial matching
          - Only supports Equal to operator
  - Multiple values with same hash code will result in slow performance
      - Storage engine will follow each row pointers in the linked list and compare the value to the lookup value.
  - Slow maintenance


Adaptive Hash Index
  - InnoDB storage engines support Adaptive Hash Index.
      - automatically
      - Can't be configure/modified
  - Built in _memory_ on the top of frequently used B-tree Indexes.
      - Evaluates query for Hash Index _and_ B-tree Index.
  - Adaptive Hash Index gives B-tree indexes very fast hashed lookups for improved performance


Other Indexes
  - Spatial Indexes
      - MyISAM
      - MySQL GIS support is not exhaustive
  - Full Text Indexes
      - Just like search engines
      - Find keywords in the text
      - They are match against operations (not **WHERE** operations)
  - Other types
      - TokuDB: Fractal Tree Index
      - ScaleDB: Patricia Tries
      - InfiniDB: Special Logic


Summary
  - Indexes reduce the random I/O of the storage system by converting it to sequential I/O whenever possible.
  - Indexes improve performance by avoiding sorting, using temporary tables and reducing additional network traffic.
