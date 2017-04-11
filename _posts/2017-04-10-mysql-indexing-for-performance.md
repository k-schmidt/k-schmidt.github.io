---
layout: post
title: "MySQL Indexing for Performance"
date: 2017-04-11
categories: MySQL
tags: MySQL
comments: true
analytics: true
---

I just learned about indexing on a MySQL database at a fairly high-level and will attempt to summarize what I've learned.
MySQL has many different kinds of indexes (full-text indexes, spatial indexes, TokuDB - fractal tree index, ScaleDB - Patricia Trie, InfiniDB - special logic) but for most use cases, InnoDB and MyISAM are the two most common.

Until MySQL 5.5, the default index was MyISAM.
MyISAM is good for analytical use cases because it has table-level locking which means two users cannot query the same table at the same time.
Because of this, MyISAM can get away with not being [ACID compliant](https://en.wikipedia.org/wiki/ACID) as well as non-transactional.
Furthermore, MyISAM doesn't support relational contraints like Foreign Keys which means that it is better to have wider tables in MyISAM rather than a [First Normal Form](https://en.wikipedia.org/wiki/Database_normalization) database.

![InnoDB vs. MyISAM]({{ site.url }}/assets/images/mysql_indexing_for_performance/innodb_vs_myisam.png)

Both InnoDB and MyISAM leverage B-tree data structures to improve query speeds.
In a B-tree index, values are stored in order and each leaf is at the same distance from the root level.
However, MyISAM and InnoDB again differ in that MyISAM stores keys and data in internal nodes whereas InnoDB only stores data in its leaf nodes with the internal nodes only containing pointers to their children.
We call the InnoDB version of B-tree a B+tree to differentiate it from the one used in MyISAM.
Due to the fact that data is only stored in the leaf nodes, InnoDB can store more indexes in memory as it traverses the tree and is therefore much smaller than a B-tree.

But you may be wondering, "Why B-trees?"
B-trees are used because it speeds up data access.
Storage engines are able to traverse the tree from the root node to the leaf node with the help of pointers even though the locations of these nodes might be spread out in the computer's memory.
We also see increased performance for full value ("Kyle Schmidt"), leftmost value or column prefix ("Kyle" from "Kyle Schmidt"), and range of values (1 to 99) query patterns as well as aiding our ORDER BY clauses because nodes point to each other in order (more to come).

![Traversing Clustered B-Tree Indexes]({{ site.url }}/assets/images/mysql_indexing_for_performance/traversing_cluster_btree_index.png)

Sometimes a single node representing a key still doesn't offer the efficiency that we need.
What if we want to store keys that are close in proximity such as employees of a certain profession?
This is where cluster indexes come in to play.
We will use cluster indexes throughout the rest of this article to talk about B-trees.
Cluster indexes have the same structure as B-trees but each "node" of a cluster B-tree contains a cluster of keys rather than a single key.

The advantages of cluster indexes are that related data is stored close to each other leading to less disk I/O while retrieving sequential or range data rather than performing a full table scan for indexes that are similar.
Furthermore, cluster indexes allow even further optimizations for data retrieval since data and indexes are stored together at leaf nodes.
Lastly, leaf nodes point to each other.
Thereby, if you perform a query on a range of data that is held within two internal cluster nodes of the cluster B-tree (in the simplest case) then the leaf nodes will be able to point to the next cluster in order without having to backtrack up the tree.

The disadvantages of a cluster index are that data insertion speed is dependent on the order of the primary key.
The table needs to be re-indexed/optimized if inserted data is not ordered by primary key which could potentially take quite a long time - time that your application might not afford to spend.
Clusters can split when new data is inserted which can lead to fragmentation of the cluster indexed B-tree structure.
All of these contribute to poor performance if the cluster index isn't re-optimized when performing an ill-adviced insertion.

B-trees support secondary indexes which further aid our queries.
All tables should have multiple secondary indexes aside from primary indexes in order to aid query optimization.
In InnoDB, the secondary index does not store actual data but only contains a pointer to the row of data.
However, if the secondary index is coupled with a primary index then the secondary index will contain a pointer to the primary index.
Secondary indexes are necessary because our queries usally make use of multiple columns of a single SQL table and therefore a single primary index will not suffice.
For example, our first query could be on a column labeled ```first_name``` and a second query on a column labeled ```last_name```.
We would need two different indexes over both of these columns in order to optimize these queries.
To carry-out our example with ```last_name``` being our secondary index and ```first_name``` as our primary index, the ```last_name``` index will **point** to the index for its corresponding ```first_name``` which will contain the data that it represents.
Similarly, a ```last_name``` only query will require the secondary key index to find the appropriate ```last_name``` and then the pointer will jump back to the primary key index before data retrieval.

![Traversing Clustered B-Tree Indexes]({{ site.url }}/assets/images/mysql_indexing_for_performance/primary_secondary_coordination.png)

Yet, there is one last type optimization that I want to talk about, Hash Indexes.
Hash indexes are built on top of hash tables and increase query performance for _exact_ lookups.
You can think of a hash as a function that when given an input computes the key of a map or dictionary-like data structure.
Hash indexes work by creating a hash value or key for each row of data.
Given an exact match of that row of data, the hash table will compute the hash of that match which 'salts' to the index of a hash table pointing to its representative row of data.
Generally, different keys generate different hash codes but if multiple values have the same hash code, then the value of the hash code in the hash table will be a linked list of row pointers.
Hash indexes are even faster than B-trees when a key doesn't contain a linked list and effective since the table resides in _memory_.

![Traversing Clustered B-Tree Indexes]({{ site.url }}/assets/images/mysql_indexing_for_performance/building_hash_index.png)

However, there are limitations to Hash indexes.
Hash indexes only contains the hash code and pointers to a row of data, _not_ the data itself.
Because it doesn't contain the original data, hash indexes are not suitable for sorting or partial matching - it can only support the equal operator.
Furthermore, I also mentioned that it is possible for multiple rows to have the same hash code, creating a linked list.
This will slow down query time considerably in these cases, creating a linear seach for a node row pointer.

But what if there is a way to leverage both the efficiency of B-trees and Hash indexes?
This is exactly what InnoDB does.
It uses what is called an Adaptive Hash Index which sits on top of a B+tree.
This comes "out-of-the-box" when you use InnoDB indexing with the caveat thast it can't be modified of configured.
The Adaptive Hash Index works by evaluating a query for its Hash Index _and_the B+tree Index, allowing us to leverage the quick lookup of hash tables and the traversal of relative nodes in a B+tree.

*Slides provided by Pluralsight.com*

<!---

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
--->
