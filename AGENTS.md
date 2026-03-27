## Build & Test Instructions

### Compile Options

#### Default Build

```bash
make clean && ./configure --enable-all
LIBRARY_PATH=/opt/homebrew/lib OPTS="-DSQLITE_ENABLE_READ_ISOLATION -DSQLITE_ENABLE_ROW_LEVEL_LOCKING" make testfixture
```

### Run All Tests

**Full test suite:**

```bash
./testfixture test/pipeline.test test/read_isolation.test test/rowlock.test  test/rowlock_read_isolation.test
```
