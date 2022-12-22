# Understand database internals

This topic will go through all the essential definitions of the database internals.

## 1. Storage concepts

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

Heap is a data structure where table (all the metadata and data) is stored.


