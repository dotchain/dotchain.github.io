# DOT Roadmap

* [Go Library](https://github.com/dotchain/dot)
  * Remaining items for v0.9.1
    * Move tests to use data files (maybe like [dataset](https://github.com/dotchain/dataset))
    * Cleanup all package documentation
  * Post v0.9.1
    * Add tree move operation
    * Add standard client
* [Go Server](https://github.com/dotchain/dots)
  * Remaining items for v0.9.1
    * Cleanup all documentation
    * Cleanup benchmark setup
    * Publish benchmark graphs
  * Possible v0.9.1 items
    * Initialize from snapshot when snapshot is available.  Avoids reading full log for the normal case.
    * Build end-to-end test suite using [dataset](https://github.com/dotchain/dataset)
    * Fetch of snapshot at version
    * Unsubscribed append of operations via Log service
  * Post v0.9.1
    * Redis support for notification
    * Redis as a storage backend
    * Delayed purging of in-memory models (currently, they get removed with last subscription)
    * Authentication, Authorization
    * Garbage collection within a model
* [ES6 Client](https://github.com/dotchain/dotjs)
  * Remaining items for v0.9.0
    * Migrate demo to heroku
    * Generic JSON model
    * Standard encodings
  * Possible v0.9.1 items
    * Grid data type
    * Graph data type
    * Rich text data type
    * Nesting data models
  * Post v0.9.1 items
    * Authentication, Authorization
    * Embedded references
