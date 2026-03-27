# Commands

```
# First build only
./configure --enable-all

# Clean build for testing
make clean && ./configure --enable-all && LIBRARY_PATH=/opt/homebrew/lib OPTS="-DSQLITE_ENABLE_READ_ISOLATION -DSQLITE_ENABLE_ROW_LEVEL_LOCKING" make testfixture

# Run the test fixture
./testfixture test/pipeline.test test/read_isolation.test test/rowlock.test  test/rowlock_read_isolation.test

# Run all tests
LIBRARY_PATH=/opt/homebrew/lib OPTS="-DSQLITE_ENABLE_READ_ISOLATION -DSQLITE_ENABLE_ROW_LEVEL_LOCKING" test/testrunner.tcl

# Other commands
make tclextension-install
gcc -DTCLSH=1 tclsqlite3.c -ltcl -lpthread -ldl -lz -lm

# Regular build
make clean && ./configure --enable-all && OPTS="-DSQLITE_ENABLE_READ_ISOLATION -DSQLITE_ENABLE_ROW_LEVEL_LOCKING" make

```
