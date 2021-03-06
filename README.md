# pg-table-observer
Observe PostgreSQL table for changes.

Requires PostgresSQL version 9.3 or above.

# Usage

```javascript
var pgp = require('pg-promise')();

import PgTableObserver from 'pg-table-observer';

const connection = 'postgres://localhost/db';

async function start() {
  try {
    let db = await pgp(connection);

    let table_observer = new PgTableObserver(db, 'myapp');

    async function cleanupAndExit() {
      await table_observer.cleanup();
      pgp.end();
      process.exit();
    }

    process.on('SIGTERM', cleanupAndExit);
    process.on('SIGINT', cleanupAndExit);

    // Multiple tables can be specified as an array

    let handle = await table_observer.notify('test', change => {
      console.log(change);
    });

    // Or trigger a callback when a condition is met

    let handle = await table_observer.trigger(condition, () => {
      console.log("condition was met")
    });

    // ... when finished observing the table

    await handle.stop();

    // ... when finished observing altogether

    await table_observer.cleanup();
  }
  catch(err) {
    console.error(err);
  }
}

// Show unhandled rejections

process.on('unhandledRejection', (err, p) => console.log(err.stack));

start();
```

# constructor(db, channel)

Parameter | Description
--------- | -----------
`db` | PostgreSQL database connection
`channel` | String, PostgreSQL LISTEN/NOTIFY channel to use, must be unique for every observer, database-wide.

# let handle = async notify(tables, callback)

Use this method if you want to observe a set of tables for any change.

The callback will be called for each INSERT, DELETE or UPDATE performed
on the tables, for each row individually.

Parameter | Description
--------- | -----------
`tables` | A string or array of tables to monitor. Table names will be converted to lowercase, and duplicates will be removed.
`callback` | function(change). Will be called for any change to any of the tables that are being monitored. See below for its fields.

Fields for the `change` parameter of the `callback`:

Field | Description
-------------- | -----------
`table` | String, name of the table that changed. ***This will always be in lowercase.***
`insert` | For INSERT, `true`
`delete` | For DELETE, `true`
`update` | For UPDATE, an object that contains the old and new values of each changed column. If a column `score` changed from 10 to 20, `change.update.score.from` would be 10 and `change.update.score.to` would be 20.
`row` | The row values, for UPDATE, the NEW row values
`old` | For UPDATE, the OLD row values

## Return value

`handle`: Object with the following fields:

Field | Description
----- | -----------
`async stop()` | async function(). Stop observing the tables.

# let handle = async trigger(tables, triggers, callback, [options])

Use this method if you want to take some action when triggered by some change to the tables.

The `triggers` callback will be called when there is a change to `tables`.
When `triggers` returns `true`, the `callback` will be called.

The following logic prevents excessive callbacks when multiple rows are updated at once:
* When `triggers` hits, `callback` is called and a timer is started.
* When `triggers` hits again before the timer is finished, the `callback` will be called as soon as the timer finishes.
* Until that happens, no `triggers` calls will happen.
* Default behavior can be changed with `options`, see below.

## Parameters

Parameter | Description
--------- | -----------
`tables` | A string or array of tables to monitor. Table names will be converted to lowercase, and duplicates will be removed.
`triggers` | function(change). Will be called when a change to `tables` happens, with the same fields as described above. If this function returns `true`, the `callback` | function(). Will be called with `triggers` returns true, as described above.
`options` | An optional object that may be used to change the default behavior as described above. See below for the possible options.

Options parameter:

Field | Description
--------------- | -----------
`trigger_first` | (default `true`): related to the 1st step above. When `true`, behaves as described above. When set to `false`, when `triggers` hit and the timer is not yet started, the `callback` will not be called immediately. Instead the timer is started and the `callback` will be called when the timer finishes. Use this if you expect many changes in short succession, or if the `callback` is relatively costly. On a heavy-load production system you may want to set this to `false`.
`trigger_delay` | (default 200ms): the time the timer will be set to. You may want to increase this value on heavy-load production system, or when the `callback` is relatively costly.
`reduce_triggers` | (default `true`): related to the 3rd step above. When `true`, behaves as described above. When `false`, `triggers` will be called for every change. The `callback` will still be called as described after the timer finishes. Use this if you want to keep track of which changes happen to which tables.


## Return value

`handle`: Object with the following fields.

Field | Description
----- | -----------
`async stop()` | async function(). Stop observing the tables.

# async cleanup()

Stops observing all tables and frees up resources.
