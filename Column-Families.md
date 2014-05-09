## Introduction

In RocksDB 3.0, we added support for Column Families.

Each key-value pair in RocksDB is associated with exactly one Column Family. If there is no Column Family specified, key-value pair is associated with Column Family "default".

Column Families provide a way to logically partition the database. Some interesting properties:
* Atomic writes across Column Families are supported. This means you can atomically execute Write({cf1, key1, value1}, {cf2, key2, value2}).
* Consistent view of the database across Column Families.
* Ability to configure different Column Families independently.
* On-the-fly adding new Column Families and dropping them. Both operations are reasonably fast.

## API

### Backward compatibility
Although we needed to make drastic API changes to support Column Families, we still support the old API. You don't need to make any changes to upgrade your application to RocksDB 3.0. All key-value pairs inserted through the old API are inserted into the Column Family "default". The same is true for downgrade after an upgrade. If you never use more than one Column Family, we don't change any disk format, which means you can safely roll back to RocksDB 2.8. This is very important for our customers inside Facebook.

### Example usage
 
    // open DB
    rocksdb::Options options;
    options.create_if_missing = true;
    rocksdb::DB* db;
    rocksdb::DB::Open(options, "test_db", &db);
    
    // create column family
    rocksdb::ColumnFamilyHandle* cf;
    rocksdb::CreateColumnFamily(rocksdb::ColumnFamilyOptions(), "new_cf", &cf);

    // close DB
    delete cf;
    delete db;

    // open DB with two column families
    std::vector<rocksdb::ColumnFamilyDescriptor> column_families;
    // have to open default column familiy
    column_families.push_back(ColumnFamilyDescriptor(rocksdb::kDefaultColumnFamilyName, rocksdb::ColumnFamilyOptions());
    // open the new one, too
    column_families.push_back(ColumnFamilyDescriptor("new_cf", rocksdb::ColumnFamilyOptions());
    std::vector<rocksdb::ColumnFamilyHandle*> handles;
    rocksdb::DB::Open(rocksdb::DBOptions(), "test_db", column_families, &handles, &db);

    // put and get from non-default column family
    db->Put(rocksdb::WriteOptions(), handles[1], Slice("key"), Slice("value")); 
    std::string value;
    db->Get(rocksdb::ReadOptions(), handles[1], Slice("key"), &value);

    // atomic write
    rocksdb::WriteBatch batch;
    batch.Put(handle[0], Slice("key2"), Slice("value2"));
    batch.Put(handle[1], Slice("key3"), Slice("value3"));
    batch.Delete(handle[0], Slice("key"));
    db->Write(rocksdb::WriteOptions(), &batch);

    // drop column family
    db->DropColumnFamily(handles[1]);

    // close db
    for (auto handle : handles) {
       delete handle;
    }
    delete db;

### Reference

#### `Options, ColumnFamilyOptions, DBOptions`

Defined in `include/rocksdb/options.h`, `Options` structures define how RocksDB behaves and performs. Before, every option was defined in a single `Options` struct. Going forward, options specific to a single Column Family will be defined in `ColumnFamilyOptions` and options specific to the whole RocksDB instance will be defined in `DBOptions`. Options struct is inheriting both ColumnFamilyOptions and DBOptions, which means you can still use it to define all the options for a DB instance with a single (default) column family.

#### `ColumnFamilyHandle`

Column Families are handled and referenced with a `ColumnFamilyHandle`. Think of it as a open file descriptor. You need to delete all `ColumnFamilyHandle`s before you delete your DB pointer. One interesting thing: Even if `ColumnFamilyHandle` is pointing to a dropped Column Family, you can continue using it. The data is actually deleted only after you delete all outstanding `ColumnFamilyHandle`s.

#### `DB::Open(const DBOptions& db_options, const std::string& name, const std::vector<ColumnFamilyDescriptor>& column_families, std::vector<ColumnFamilyHandle*>* handles, DB** dbptr);`

When opening a DB in a read-write mode, you need to specify all Column Families that currently exist in a DB. If that's not the case, `DB::Open` call will return `Status::InvalidArgument()`.  You specify Column Families with a vector of `ColumnFamilyDescriptor`s. `ColumnFamilyDescriptor` is just a struct with a Column Family name and `ColumnFamilyOptions`. Open call will return a `Status` and also a vector of pointers to `ColumnFamilyHandle`s, which you can then use to reference Column Families. Make sure to delete all `ColumnFamilyHandle`s before you delete the DB pointer.

#### `DB::OpenForReadOnly(const DBOptions& db_options, const std::string& name, const std::vector<ColumnFamilyDescriptor>& column_families, std::vector<ColumnFamilyHandle*>* handles, DB** dbptr, bool error_if_log_file_exist = false)`

The behavior is similar to `DB::Open`, except that it opens DB in read-only mode. One big difference is that when opening the DB as read-only, you don't need to specify all Column Families -- you can only open a subset of Column Families.

#### `DB::ListColumnFamilies(const DBOptions& db_options, const std::string& name, std::vector<std::string>* column_families)`

`ListColumnFamilies` is a static function that returns the list of all column families currently present in the DB.

#### `DB::CreateColumnFamily(const ColumnFamilyOptions& options, const std::string& column_family_name, ColumnFamilyHandle** handle)`

Creates a Column Family specified with option and a name and returns `ColumnFamilyHandle` through an argument.

#### `DropColumnFamily(ColumnFamilyHandle* column_family)`

Drop the column family specified by `ColumnFamilyHandle`. Note that the actual data is not deleted until the client calls `delete column_family;`. You can still continue using the column family if you have outstanding `ColumnFamilyHandle` pointer.

#### `DB::NewIterators(const ReadOptions& options, const std::vector<ColumnFamilyHandle*>& column_families, std::vector<Iterator*>* iterators)`

This is the new call, which enables you to create iterators on multiple Column Families that have consistent view of the database.

#### WriteBatch

To execute multiple writes atomically, you need to build a `WriteBatch`. All `WriteBatch` API calls now also take `ColumnFamilyHandle*` to specify the Column Family you want to write to.

#### All other API calls

All other API calls have a new argument `ColumnFamilyHandle*`, through which you can specify the Column Family.

## Implementation

The main idea behind Column Families is that they share the write-ahead log and don't share memtables and table files. By sharing write-ahead logs we get awesome benefit of atomic writes. By separating memtables and table files, we are able to configure column families independently and delete them quickly.

Every time a single Column Family is flushed, we create a new WAL (write-ahead log). All new writes to all Column Families go to the new WAL. However, we still can't delete the old WAL since it contains live data from other Column Families. We can delete the old WAL only when all Column Families have been flushed and all data contained in that WAL persisted in table files. This created some interesting implementation details and will create interesting tuning requirements. Make sure to tune your RocksDB such that all column families are regularly flushed. Also, take a look at `Options::max_total_wal_size`, which can be configured such that stale column families are automatically flushed.