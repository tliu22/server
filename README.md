# MDEV 18188: Maintain persistent COUNT(*) in InnoDB
### [Objective](https://jira.mariadb.org/browse/MDEV-18188)
>Query planning needs to know the number of records in a table. Currently, InnoDB only provides an estimate of this.
If InnoDB kept accurate track of the number of records in a table, then it would not only benefit statistics, but also limited cases like \[simple COUNT(*) queries\].

This implementation intends to persistently store a count of a table's committed records within the clustered index (see [Committed Count Storage](#Committed-Count-Storage)). Meanwhile, it will ephemerally store a count of each table's uncommitted records within the `trx_t` struct for each transaction.The uncommitted count is the difference between the committed `COUNT(*)` and the transaction's read view `COUNT(*)`.

This implementation is only applicable for the READ COMMITTED isolation level.

### Committed Count Storage
The committed count for a table will be stored at the offset of `4 + 2 * num_non_pk_fields` bytes within `mblob` of the clustered index's hidden metadata record, following the `INSTANT DROP/ADD` columns that are also stored in `mblob`.

### API
`dict0mem.h`:
```c
// Committed count (per primary index)
struct dict_index_t {
    ...
    bool is_init_committed_count();
    void init_committed_count(ulint init_count);
    void add_committed_count(lint diff_count);
    ulint read_committed_count();
    ...
}
```
`trx0trx.h`:
```c
// Uncommitted count (per transaction and table)
struct trx_t {
    ...
    void add_uncommitted_count(dict_table_t* table, lint diff_count); /* If uncommitted count for table is not yet initialized, 
                                                                       * it will be initialized with value 0 */
    lint read_uncommitted_count(dict_table_t* table);  /* Returns 0 if no writes to table have been done */
    ...
}
```

### Functions to Change
```c
// Open table
int ha_innobase::open(...) {
    ...
    if (clustered_index->is_init_committed_count()) {
        committed_count = clustered_index->read_committed_count();
    } else {
        // Calculate COUNT(*) into num_rows
        ...
        clustered_index->init_committed_count(num_rows);
    }
    ...
}
``` 
```c
// SELECT COUNT(*)
int ha_innobase::check(...) {
    ...
    *n_rows = index->read_committed_count() 
              + trx->read_uncommitted_count(table);
    ...
}
```
```c
// INSERT
int ha_innobase::write_row(...) {
    ...
    trx->add_uncommitted_count(num_inserted_rows);
    ...
}
```
```c
// DELETE
int ha_innobase::delete_row(...) { 
    ...
    trx->add_uncommitted_count((-1) * num_deleted_rows);
    ...
}
```
```c
// COMMIT
static int innobase_commit(...) {
    ...
    for modified_table in modified_tables {
        modified_table->clustered_index->add_committed_count(
            trx->read_uncommitted_count(modified_table)
        );
    }
    ...
}
```

### Synchronization
Committed count functions will be synchronized by a lock. Uncommitted count functions will be implemented with an atomic integer for the count.
