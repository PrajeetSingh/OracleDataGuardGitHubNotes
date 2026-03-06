# Data Guard Far Sync Architecture (19c)

`Technical notes, configuration steps, and troubleshooting for Oracle DataGuard.`

When we move data over long distances, like from one province to another, we usually have to choose between speed or safety. If we turn on SYNC, the Primary database becomes slow because it's waiting for the remote side. If we use ASYNC, we risk losing data if the main site goes down.

To take care of above issues, I'm using a Far Sync setup.

## Why we need this

The idea is simple: we put a small "Far Sync" instance close to our Primary database.

* **The Primary** sends data to Far Sync using SYNC. Since they are close, there is no lag.

* **Far Sync** then pushes that data to our remote Standby using ASYNC.

This way, we get Zero Data Loss without making the Primary slow.

## Key Benefits

* **No Lag:** The application doesn't feel any slowness.

* **Safe Data:** Even if the Primary crashes, the data is already safe in Far Sync.

* **Low Cost:** The Far Sync VM only needs 2 cores and 8GB RAM because it doesn't store actual data files, only redo logs.