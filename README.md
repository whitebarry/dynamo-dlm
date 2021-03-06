# dynamo-dlm
[![Build Status](https://travis-ci.org/barrybarrette/dynamo-dlm.svg?branch=master)](https://travis-ci.org/barrybarrette/dynamo-dlm)
[![Coverage Status](https://coveralls.io/repos/github/whitebarry/dynamo-dlm/badge.svg)](https://coveralls.io/github/whitebarry/dynamo-dlm)


Distributed lock manager for Python using AWS DynamoDB for persistence

Currently this module exposes a single distributed locking primitive, `DynamoDbLock` that functions similarly to `threading.Lock` in the standard library 

Locks are scoped to a logical resource, represented by an arbitrary but uniquely identifying string, referred to below as the `resource_id`.
All instances of `DynamoDbLock` with the same table name and resource id will respect the lock rules. 
By default, the lock looks for a DynamoDB table named `dynamo_dlm_locks`. 
You may use a custom name for the table as outlined below.
The table must have a primary key of type `String` named `resource_id` and NO sort key:


![](https://i.imgur.com/vOgVsAd.png)

Your application will need the following permissions in order to function properly:
![](https://i.imgur.com/nw5F1ab.png)

##### A note on capacity and performance:
The lock class has been designed such that it should never fail under normal circumstances.
Given enough time, it should eventually acquire a lock even if there are thousands of simultaneous requests.
When using on-demand capacity all locks should acquire in constant time, every time.   
DynamoDB provisioned capacity is a topic too in depth to go into here but if you are using it and run into issues with locks taking too long to acquire then you likely need to increase your write capacity.
The lock should **never** consume read capacity. This is due to the way that DynamoDB handles conditional writes.
Logging has been added at log level `WARNING` to notify you of any backoffs related to provisioned capacity.  
When not constrained by write capacity or network speed, the average acquire/release cycle takes approximately 100ms when running outside of AWS. 
If your execution environment is within AWS it will be markedly faster, as low as 10 ms when within the same region.

## Installation and usage

Install via pip:

`pip install dynamo-dlm`


Using the lock primitive is pretty straightforward. 
Just create an instance with a unique identifier and call `acquire()`. 
This will block any other instances' calls to `acquire()` until `release()` is called or the lock expires, whichever comes first.
When you're done with the resource you needed locked, call `release()` to give up control of the lock.
Remember: all instances with the same identifier and table name will respect the same lock.
Ideally you want to time the expiration to slightly longer than you expect the operation to take.
The lock only operates on whole second increments, so the shortest reasonable lock time is 1 second.
As you're developing, keep an eye on the `WARNING` logger for indications that your locks are expiring, meaning operations are taking longer than expected. 

```python
import dynamo_dlm as dlm

resource_id = 'a unique resource identifier'
lock = dlm.DynamoDbLock(resource_id)
lock.acquire()
# Code executed here is protected by the lock until it expires
lock.release()
```

The lock class is also implemented as a context manager:
```python
import dynamo_dlm as dlm

resource_id = 'a unique resource identifier'
with dlm.DynamoDbLock(resource_id):
    # Code executed inside the "with" block is protected by the lock until it expires 
    pass
```

By default, locks last for 10 seconds if not released.
The duration and/or table name can be set globally at the module level:
```python
import dynamo_dlm as dlm

dlm.DEFAULT_DURATION = 5
dlm.DEFAULT_TABLE_NAME = 'my_dynamo_db_lock_table'

resource_id = 'a unique resource identifier'
lock = dlm.DynamoDbLock(resource_id)
```

They can also be overridden per lock:
```python
import dynamo_dlm as dlm

resource_id = 'a unique resource identifier'
lock = dlm.DynamoDbLock(resource_id, duration=5, table_name='my_dynamo_db_lock_table')
```
