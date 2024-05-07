## Appendix B. Altibase Limitations

### Maximum Altibase Values

The following table lists the maximum possible values that various Altibase objects can have. 


|Identification	|Maximum Values of Objects	|Remarks|
|---------------|---------------------------|-------|
|DB_NAME Length	|128	|The maximum length of the database name|
|OBJECT Length	|40	|The maximum length of database object names|
|The Number of Tablespaces	|64K (Including System Tablespaces)	|The maximum number of tablespaces in one DB|
|The Number of Data Files	|1,024	|The maximum number of data files in one tablespace|
||67,108,864 (64K * 1024)	|The maximum number of data files in one DB|
|The Number of Users	|2,147,483,638 (Including System Users)	|The maximum number of users in one DB|
|The Number of Tables	|2,097,151(Including Meta Tables)	|The maximum number of tables in one DB|
|The Number of Indexes	|64	|The maximum number of indexes per table|
|The Number of Columns	|1,024	|The maximum number of columns per table|
||32	|The maximum number of key columns per index|
|The Number of Rows	|Unlimited	|The maximum number of row per table|
|The Number of Partitions	|2,147,483,638 (In the Entire System)	|The maximum number of partitions in one DB|
|The Number of Constraints	|2,147,483,638 (In the Entire System)	|The maximum number of constraints in one DB|