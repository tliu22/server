# MDEV 18188: Maintain persistent COUNT(*) in InnoDB
### [Objective](https://jira.mariadb.org/browse/MDEV-18188)
>Query planning needs to know the number of records in a table. Currently, InnoDB only provides an estimate of this.
If InnoDB kept accurate track of the number of records in a table, then it would not only benefit statistics, but also limited cases like \[simple COUNT(*) queries\].

This implementation intends to persistently store a count of a table's committed records within the clustered index (see [Committed Count Storage](#Committed-Count-Storage)). Meanwhile, it will ephemerally store a count of each table's uncommitted records within the `trx_t` struct for each transaction.The uncommitted count is the difference between the committed `COUNT(*)` and the transaction's read view `COUNT(*)`.

This implementation is only applicable for the READ COMMITTED isolation level.

### Committed Count Storage
Create a committed count hidden metadata record with `info_bits &= (REC_INFO_MIN_REC_FLAG | REC_INFO_DELETED_FLAG)`. This record will contain a metadata BLOB pointer. To distinguish the committed count hidden metadata record from the `INSTANT` hidden metadata record, the first 4 bytes in the former's BLOB will be set to `REC_MAX_N_FIELDS` (from `rem0types.h`), since the first 4 bytes in the latter's BLOB will be interpreted as `num_non_pk_fields`, which should always be less than `REC_MAX_N_FIELDS - 3`. The next 8 bytes of the BLOB represent the table's persisted committed count in the form of an 64-bit unsigned integer.

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
    /* If uncommitted count for table is not yet initialized, 
     * it will be initialized with value 0 */
    void add_uncommitted_count(dict_table_t* table, lint diff_count); 
    /* Returns 0 if no writes to table have been done */
    lint read_uncommitted_count(dict_table_t* table);  
    ...
}
```

### Functions to Change
`ha_innodb.cc`:
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

// SELECT COUNT(*)
int ha_innobase::check(...) {
    ...
    *n_rows = index->read_committed_count() 
              + trx->read_uncommitted_count(table);
    ...
}

// INSERT
int ha_innobase::write_row(...) {
    ...
    trx->add_uncommitted_count(num_inserted_rows);
    ...
}

// DELETE
int ha_innobase::delete_row(...) { 
    ...
    trx->add_uncommitted_count((-1) * num_deleted_rows);
    ...
}

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

`btr0cur.cc`:
```c
// Check next 
static dberr_t btr_cur_instant_init_low(...) {
  ...
  if (info_bits & (REC_INFO_MIN_REC_FLAG | REC_INFO_DELETED_FLAG)) {
      if (first 4 bytes after headers == REC_MAX_N_FIELDS) {
          // Current record is committed count metadata record, move onto next
          page_cur_move_to_next(&cur.page_cur); 
          ...
      } 
  }
  ...
}
```

### Synchronization
Committed count functions will be synchronized by a lock. Uncommitted count functions will be implemented with an atomic integer for the count.
