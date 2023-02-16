# Understand database internals

This topic will go through all the essential definitions of the database internals.

### Pages

Pages are fixed size memory locations. Rows of table are stored in these logical pages.

Database doesn't read a single row, it will read a page (or more), that contains the row you need, in a single IO, and database will get a lot of rows in that page, after that, database will filter and return to you the row you need.

The size of page will be determined by the DBMS (Example: 8Kb in Postgres, 16KB in MySQL). The number of rows stored in page will be determined by the size of page and the size of content in each row.

<p align="center">
    <img src="https://raw.githubusercontent.com/tienmanh2208/public-asset-depressed-developer/master/posts/database/fundamental_definition/page_sql.png" alt="drawing" style="width:300px;"/>
</p>

Assume that each page will hold 3 rows, if you have 1001 rows, you will have 334 pages (page 0 to page 333)

### IO

IO operations is a read request to the disk to get page(s).

We need to minimize this operation as much as possible because this operation is expensive.

An IO can fetch 1 page or more depending on the disk partitions and other factors (like query result).

An IO can not get a single row. It can get a single page with many rows in them (that contains the needed row).

Some IOs in operating system goes to operating system cache to get page (not disk).

### Heap

Heap is a data structure where table (all the metadata and data) is stored. This is where the actual data is stored including everything.

Traversing the heap is expensive as we need to read so many data to get what we want. That's why we need indexes, that will tell us exactly which part of the heap we need to read, what page we need to pull from heap.

### Index

An index is another data structure (separate from the heap) that has `pointers` to the heap.

It has part of the data, and be used to quickly search for something.

You can index on one column or more.

Assume you have a value and you want to find the row has that value in database, if you don't index the table, you have to go through all the pages of the heap to find that value. But, if you indexed your table, you can find that value in index first, once you find the value you are looking for in the index, you can go to the heap and get all the related information where everything is there. Index will tell you exactly which page to fetch in the heap instead of scan every page in the heap to get data.

Index is also stored as pages and cost IO to pull the entries of the index. **The smaller the index is, the more it can fit in the memory, the faster the search can be.**

Popular data structure for index is B-Tree.

*The image bellow ilustrates flow search data of database when you search with indexed column.*

<p align="center">
    <img src="https://raw.githubusercontent.com/tienmanh2208/public-asset-depressed-developer/master/posts/database/fundamental_definition/flow_searching_with_index.png" alt="drawing" style="width:100%;"/>
</p>
