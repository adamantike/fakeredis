fakeredis: A fake version of a redis-py
=======================================

.. image:: https://secure.travis-ci.org/jamesls/fakeredis.svg?branch=master
   :target: http://travis-ci.org/jamesls/fakeredis


.. image:: https://coveralls.io/repos/jamesls/fakeredis/badge.svg?branch=master
   :target: https://coveralls.io/r/jamesls/fakeredis


fakeredis is a pure-Python implementation of the redis-py python client
that simulates talking to a redis server.  This was created for a single
purpose: **to write unittests**.  Setting up redis is not hard, but
many times you want to write unittests that do not talk to an external server
(such as redis).  This module now allows tests to simply use this
module as a reasonable substitute for redis.

Although fakeredis is pure Python, you will need lupa_ if you want to run Lua
scripts. If you install fakeredis with ``pip install fakeredis[lua]`` it will
be automatically installed.

.. _lupa: https://pypi.org/project/lupa/

Alternatives
============

Consider using birdisle_ instead of fakeredis. It embeds the redis codebase
into a Python extension, so it implements the full redis command set and
behaves far more closely to a real redis implementation. The disadvantage is
that it currently only works on Linux.

.. _birdisle: https://birdisle.readthedocs.io/en/latest/


How to Use
==========

The intent is for fakeredis to act as though you're talking to a real
redis server.  It does this by storing state internally.
For example:

.. code-block:: python

  >>> import fakeredis
  >>> r = fakeredis.FakeStrictRedis()
  >>> r.set('foo', 'bar')
  True
  >>> r.get('foo')
  'bar'
  >>> r.lpush('bar', 1)
  1
  >>> r.lpush('bar', 2)
  2
  >>> r.lrange('bar', 0, -1)
  [2, 1]

The state is stored in an instance of `FakeServer`. If one is not provided at
construction, a new instance is automatically created for you, but you can
explicitly create one to share state:

.. code-block:: python

  >>> import fakeredis
  >>> server = fakeredis.FakeServer()
  >>> r1 = fakeredis.FakeStrictRedis(server=server)
  >>> r1.set('foo', 'bar')
  True
  >>> r2 = fakeredis.FakeStrictRedis(server=server)
  >>> r2.get('foo')
  'bar'
  >>> r2.set('bar', 'baz')
  True
  >>> r1.get('bar')
  'baz'
  >>> r2.get('bar')
  'baz'

It is also possible to mock connection errors so you can effectively test
your error handling. Simply set the connected attribute of the server to
`False` after initialization.

.. code-block:: python

  >>> import fakeredis
  >>> server = fakeredis.FakeServer()
  >>> server.connected = False
  >>> r = fakeredis.FakeStrictRedis(server=server)
  >>> r.set('foo', 'bar')
  ConnectionError: FakeRedis is emulating a connection error.
  >>> server.connected = True
  >>> r.set('foo', 'bar')
  True

Fakeredis implements the same interface as `redis-py`_, the
popular redis client for python, and models the responses
of redis 5.0.

Porting to fakeredis 1.0
========================

Version 1.0 is an almost total rewrite, intended to support redis-py 3.x and
improve the Lua scripting emulation. It has a few backwards incompatibilities
that may require changes to your code:

1. By default, each FakeRedis or FakeStrictRedis instance contains its own
   state. This is equivalent to the `singleton=True` option to previous
   versions of fakeredis. This change was made to improve isolation between
   tests. If you need to share state between instances, create a FakeServer,
   as described above.

2. FakeRedis is now a subclass of FakeStrictRedis, and similarly
   FakeStrictRedis is a subclass of StrictRedis. Code that uses `isinstance`
   may behave differently.

3. The `connected` attribute is now a property of `FakeServer`, rather than
   `FakeRedis` or `FakeStrictRedis`. You can still pass the property to the
   constructor of the latter (provided no server is provided).



Unimplemented Commands
======================

All of the redis commands are implemented in fakeredis with
these exceptions:


connection
----------

 * auth
 * quit


server
------

 * bgrewriteaof
 * client id
 * client kill
 * client list
 * client getname
 * client pause
 * client reply
 * client setname
 * client unblock
 * command
 * command count
 * command getkeys
 * command info
 * config get
 * config rewrite
 * config set
 * config resetstat
 * debug object
 * debug segfault
 * info
 * memory doctor
 * memory help
 * memory malloc-stats
 * memory purge
 * memory stats
 * memory usage
 * monitor
 * role
 * shutdown
 * slaveof
 * replicaof
 * slowlog
 * sync
 * time


string
------

 * bitfield
 * bitop
 * bitpos


sorted_set
----------

 * bzpopmin
 * bzpopmax
 * zpopmax
 * zpopmin


cluster
-------

 * cluster addslots
 * cluster count-failure-reports
 * cluster countkeysinslot
 * cluster delslots
 * cluster failover
 * cluster forget
 * cluster getkeysinslot
 * cluster info
 * cluster keyslot
 * cluster meet
 * cluster nodes
 * cluster replicate
 * cluster reset
 * cluster saveconfig
 * cluster set-config-epoch
 * cluster setslot
 * cluster slaves
 * cluster replicas
 * cluster slots
 * readonly
 * readwrite


generic
-------

 * dump
 * migrate
 * object
 * restore
 * touch
 * wait


geo
---

 * geoadd
 * geohash
 * geopos
 * geodist
 * georadius
 * georadiusbymember


pubsub
------

 * pubsub


scripting
---------

 * script debug
 * script exists
 * script flush
 * script kill


stream
------

 * xinfo
 * xadd
 * xtrim
 * xdel
 * xrange
 * xrevrange
 * xlen
 * xread
 * xgroup
 * xreadgroup
 * xack
 * xclaim
 * xpending


Other limitations
=================

Apart from unimplemented commands, there are a number of cases where fakeredis
won't give identical results to real redis. The following are differences that
are unlikely to ever be fixed; there are also differences that are fixable
(such as commands that do not support all features) which should be filed as
bugs in Github.

1. Hyperloglogs are implemented using sets underneath. This means that the
   `type` command will return the wrong answer, you can't use `get` to retrieve
   the encoded value, and counts will be slightly different (they will in fact be
   exact).

2. When a command has multiple error conditions, such as operating on a key of
   the wrong type and an integer argument is not well-formed, the choice of
   error to return may not match redis.

3. The `incrbyfloat` and `hincrbyfloat` commands in redis use the C `long
   double` type, which typically has more precision than Python's `float`
   type.

4. Redis makes guarantees about the order in which clients blocked on blocking
   commands are woken up. Fakeredis does not honour these guarantees.

5. Where redis contains bugs, fakeredis generally does not try to provide exact
   bug-compatibility. It's not practical for fakeredis to try to match the set
   of bugs in your specific version of redis.

6. There are a number of cases where the behaviour of redis is undefined, such
   as the order of elements returned by set and hash commands. Fakeredis will
   generally not produce the same results, and in Python versions before 3.6
   may produce different results each time the process is re-run.

7. SCAN/ZSCAN/HSCAN/SSCAN will not necessary iterate all items if items are
   deleted or renamed during iteration. They also won't necessarily iterate in
   the same chunk sizes or the same order as redis.


Contributing
============

Contributions are welcome.  Please see the `contributing guide`_ for
more details. The maintainer generally has very little time to work on
fakeredis, so the best way to get a bug fixed is to contribute a pull
request.

If you'd like to help out, you can start with any of the issues
labeled with `HelpWanted`_.


Running the Tests
=================

To ensure parity with the real redis, there are a set of integration tests
that mirror the unittests.  For every unittest that is written, the same
test is run against a real redis instance using a real redis-py client
instance.  In order to run these tests you must have a redis server running
on localhost, port 6379 (the default settings).  The integration tests use
db=10 in order to minimize collisions with an existing redis instance.


To run all the tests, install the requirements file::

    pip install -r requirements.txt

If you just want to run the unittests::

    nosetests test_fakeredis.py:TestFakeStrictRedis test_fakeredis.py:TestFakeRedis

Because this module is attempting to provide the same interface as `redis-py`_,
the python bindings to redis, a reasonable way to test this to to take each
unittest and run it against a real redis server.  fakeredis and the real redis
server should give the same result.  This ensures parity between the two.  You
can run these "integration" tests like this::

    nosetests test_fakeredis.py:TestRealStrictRedis test_fakeredis.py:TestRealRedis test_fakeredis_hypothesis.py

In terms of implementation, ``TestRealRedis`` is a subclass of
``TestFakeRedis`` that overrides a factory method to create
an instance of ``redis.Redis`` (an actual python client for redis)
instead of ``fakeredis.FakeStrictRedis``.

To run both the unittests and the "integration" tests, run::

    nosetests

If redis is not running and you try to run tests against a real redis server,
these tests will have a result of 'S' for skipped.

There are some tests that test redis blocking operations that are somewhat
slow.  If you want to skip these tests during day to day development,
they have all been tagged as 'slow' so you can skip them by running::

    nosetests -a '!slow' test_fakeredis.py


Revision history
================

1.0.5
-----
- `#247 <https://github.com/jamesls/fakeredis/pull/247>`_ Support NX/XX/CH flags in ZADD command
- `#250 <https://github.com/jamesls/fakeredis/pull/250>`_ Implement UNLINK command
- `#252 <https://github.com/jamesls/fakeredis/pull/252>`_ Fix implementation of ZSCAN

1.0.4
-----
- `#240 <https://github.com/jamesls/fakeredis/issues/240>`_ `#242 <https://github.com/jamesls/fakeredis/issues/242>`_ Support for ``redis==3.3``

1.0.3
-----
- `#235 <https://github.com/jamesls/fakeredis/issues/235>`_ Support for ``redis==3.2``

1.0.2
-----
- `#235 <https://github.com/jamesls/fakeredis/issues/235>`_ Depend on ``redis<3.2``

1.0.1
-----
- Fix crash when a connection closes without unsubscribing and there is a subsequent PUBLISH

1.0
---

Version 1.0 is a major rewrite. It works at the redis protocol level, rather
than at the redis-py level. This allows for many improvements and bug fixes.

- `#225 <https://github.com/jamesls/fakeredis/issues/225>`_ Support redis-py 3.0
- `#65 <https://github.com/jamesls/fakeredis/issues/65>`_ Support `execute_command` method
- `#206 <https://github.com/jamesls/fakeredis/issues/206>`_ Drop Python 2.6 support
- `#141 <https://github.com/jamesls/fakeredis/issues/141>`_ Support strings in integer arguments
- `#218 <https://github.com/jamesls/fakeredis/issues/218>`_ Watches checks commands rather than final value
- `#220 <https://github.com/jamesls/fakeredis/issues/220>`_ Better support for calling into redis from Lua
- `#158 <https://github.com/jamesls/fakeredis/issues/158>`_ Better timestamp handling
- Support for `register_script` function.
- Fixes for race conditions caused by keys expiring mid-command
- Disallow certain commands in scripts
- Fix handling of blocking commands inside transactions
- Fix handling of PING inside pubsub connections

It also has new unit tests based on hypothesis_, which has identified many
corner cases that are now handled correctly.

.. _hypothesis: https://hypothesis.readthedocs.io/en/latest/

1.0rc1
------
Compared to 1.0b1:

- `#231 <https://github.com/jamesls/fakeredis/pull/231>`_ Fix setup.py, fakeredis is directory/package now
- Fix some corner case handling of +0 vs -0
- Fix pubsub `get_message` with a timeout
- Disallow certain commands in scripts
- Fix handling of blocking commands inside transactions
- Fix handling of PING inside pubsub connections
- Make hypothesis tests skip if redis is not running
- Minor optimisations to zset

1.0b1
-----
Version 1.0 is a major rewrite. It works at the redis protocol level, rather
than at the redis-py level. This allows for many improvements and bug fixes.

- `#225 <https://github.com/jamesls/fakeredis/issues/225>`_ Support redis-py 3.0
- `#65 <https://github.com/jamesls/fakeredis/issues/65>`_ Support `execute_command` method
- `#206 <https://github.com/jamesls/fakeredis/issues/206>`_ Drop Python 2.6 support
- `#141 <https://github.com/jamesls/fakeredis/issues/141>`_ Support strings in integer arguments
- `#218 <https://github.com/jamesls/fakeredis/issues/218>`_ Watches checks commands rather than final value
- `#220 <https://github.com/jamesls/fakeredis/issues/220>`_ Better support for calling into redis from Lua
- `#158 <https://github.com/jamesls/fakeredis/issues/158>`_ Better timestamp handling
- Support for `register_script` function.
- Fixes for race conditions caused by keys expiring mid-command

It also has new unit tests based on hypothesis_, which has identified many
corner cases that are now handled correctly.

.. _hypothesis: https://hypothesis.readthedocs.io/en/latest/

0.16.0
------
- `#224 <https://github.com/jamesls/fakeredis/pull/224>`_ Add __delitem__
- Restrict to redis<3

0.15.0
------
- `#219 <https://github.com/jamesls/fakeredis/pull/219>`_ Add SAVE, BGSAVE and LASTSAVE commands
- `#222 <https://github.com/jamesls/fakeredis/pull/222>`_ Fix deprecation warnings in Python 3.7

0.14.0
------
This release greatly improves support for threads: the bulk of commands are now
thread-safe, ``lock`` has been rewritten to more closely match redis-py, and
pubsub now supports ``run_in_thread``:

- `#213 <https://github.com/jamesls/fakeredis/issues/217>`_ pipeline.watch runs transaction even if no commands are queued
- `#214 <https://github.com/jamesls/fakeredis/pull/214>`_ Added pubsub.run_in_thread as it is implemented in redis-py
- `#215 <https://github.com/jamesls/fakeredis/pull/215>`_ Keep pace with redis-py for zrevrange method
- `#216 <https://github.com/jamesls/fakeredis/pull/216>`_ Update behavior of lock to behave closer to redis lock

0.13.1
------
- `#208 <https://github.com/jamesls/fakeredis/pull/208>`_ eval's KEYS and ARGV are now lua tables
- `#209 <https://github.com/jamesls/fakeredis/pull/209>`_ Redis operation that returns dict now converted to Lua table when called inside eval operation
- `#212 <https://github.com/jamesls/fakeredis/pull/212>`_ Optimize ``_scan()``

0.13.0.1
--------
- Fix a typo in the Trove classifiers

0.13.0
------
- `#202 <https://github.com/jamesls/fakeredis/pull/202>`_ Function smembers returns deepcopy
- `#205 <https://github.com/jamesls/fakeredis/pull/205>`_ Implemented hstrlen
- `#207 <https://github.com/jamesls/fakeredis/pull/207>`_ Test on Python 3.7

0.12.0
------
- `#197 <https://github.com/jamesls/fakeredis/pull/197>`_ Mock connection error
- `#195 <https://github.com/jamesls/fakeredis/pull/195>`_ Align bool/len behaviour of pipeline
- `#199 <https://github.com/jamesls/fakeredis/issues/199>`_ future.types.newbytes does not encode correctly

0.11.0
------
- `#194 <https://github.com/jamesls/fakeredis/pull/194>`_ Support ``score_cast_func`` in zset functions
- `#192 <https://github.com/jamesls/fakeredis/pull/192>`_ Make ``__getitem__`` raise a KeyError for missing keys

0.10.3
------
This is a minor bug-fix release.

- `#189 <https://github.com/jamesls/fakeredis/pull/189>`_ Add 'System' to the list of libc equivalents

0.10.2
------
This is a bug-fix release.

- `#181 <https://github.com/jamesls/fakeredis/issues/181>`_ Upgrade twine & other packaging dependencies
- `#106 <https://github.com/jamesls/fakeredis/issues/106>`_ randomkey method is not implemented, but is not in the list of unimplemented commands
- `#170 <https://github.com/jamesls/fakeredis/pull/170>`_ Prefer readthedocs.io instead of readthedocs.org for doc links
- `#180 <https://github.com/jamesls/fakeredis/issues/180>`_ zadd with no member-score pairs should fail
- `#145 <https://github.com/jamesls/fakeredis/issues/145>`_ expire / _expire: accept 'long' also as time
- `#182 <https://github.com/jamesls/fakeredis/issues/182>`_ Pattern matching does not match redis behaviour
- `#135 <https://github.com/jamesls/fakeredis/issues/135>`_ Scan includes expired keys
- `#185 <https://github.com/jamesls/fakeredis/issues/185>`_ flushall() doesn't clean everything
- `#186 <https://github.com/jamesls/fakeredis/pull/186>`_ Fix psubscribe with handlers
- Run CI on PyPy
- Fix coverage measurement

0.10.1
------
This release merges the fakenewsredis_ fork back into fakeredis. The version
number is chosen to be larger than any fakenewsredis release, so version
numbers between the forks are comparable. All the features listed under
fakenewsredis version numbers below are thus included in fakeredis for the
first time in this release.

Additionally, the following was added:
- `#169 <https://github.com/jamesls/fakeredis/pull/169>`_ Fix set-bit

fakenewsredis 0.10.0
--------------------
- `#14 <https://github.com/ska-sa/fakenewsredis/pull/14>`_ Add option to create an instance with non-shared data
- `#13 <https://github.com/ska-sa/fakenewsredis/pull/13>`_ Improve emulation of redis -> Lua returns
- `#12 <https://github.com/ska-sa/fakenewsredis/pull/12>`_ Update tox.ini: py35/py36 and extras for eval tests
- `#11 <https://github.com/ska-sa/fakenewsredis/pull/11>`_ Fix typo in private method name

fakenewsredis 0.9.5
-------------------
This release makes a start on supporting Lua scripting:
- `#9 <https://github.com/ska-sa/fakenewsredis/pull/9>`_ Add support for StrictRedis.eval for Lua scripts

fakenewsredis 0.9.4
-------------------
This is a minor bugfix and optimization release:
- `#5 <https://github.com/ska-sa/fakenewsredis/issues/5>`_ Update to match redis-py 2.10.6
- `#7 <https://github.com/ska-sa/fakenewsredis/issues/7>`_ Set with invalid expiry time should not set key
- Avoid storing useless expiry times in hashes and sorted sets
- Improve the performance of bulk zadd

fakenewsredis 0.9.3
-------------------
This is a minor bugfix release:
- `#6 <https://github.com/ska-sa/fakenewsredis/pull/6>`_ Fix iteration over pubsub list
- `#3 <https://github.com/ska-sa/fakenewsredis/pull/3>`_ Preserve expiry time when mutating keys
- Fixes to typos and broken links in documentation

fakenewsredis 0.9.2
-------------------
This is the first release of fakenewsredis, based on fakeredis 0.9.0, with the following features and fixes:

- fakeredis `#78 <https://github.com/jamesls/fakeredis/issues/78>`_ Behaviour of transaction() does not match redis-py
- fakeredis `#79 <https://github.com/jamesls/fakeredis/issues/79>`_ Implement redis-py's .lock()
- fakeredis `#90 <https://github.com/jamesls/fakeredis/issues/90>`_ HINCRBYFLOAT changes hash value type to float
- fakeredis `#101 <https://github.com/jamesls/fakeredis/issues/101>`_ Should raise an error when attempting to get a key holding a list)
- fakeredis `#146 <https://github.com/jamesls/fakeredis/issues/146>`_ Pubsub messages and channel names are forced to be ASCII strings on Python 2
- fakeredis `#163 <https://github.com/jamesls/fakeredis/issues/163>`_ getset does not to_bytes the value
- fakeredis `#165 <https://github.com/jamesls/fakeredis/issues/165>`_ linsert implementation is incomplete
- fakeredis `#128 <https://github.com/jamesls/fakeredis/pull/128>`_ Remove `_ex_keys` mapping
- fakeredis `#139 <https://github.com/jamesls/fakeredis/pull/139>`_ Fixed all flake8 errors and added flake8 to Travis CI
- fakeredis `#166 <https://github.com/jamesls/fakeredis/pull/166>`_ Add type checking
- fakeredis `#168 <https://github.com/jamesls/fakeredis/pull/168>`_ Use repr to encode floats in to_bytes

.. _fakenewsredis: https://github.com/ska-sa/fakenewsredis
.. _redis-py: http://redis-py.readthedocs.io/
.. _contributing guide: https://github.com/jamesls/fakeredis/blob/master/CONTRIBUTING.rst
.. _HelpWanted: https://github.com/jamesls/fakeredis/issues?q=is%3Aissue+is%3Aopen+label%3AHelpWanted
