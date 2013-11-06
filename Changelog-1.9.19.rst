uWSGI 1.9.19
============

Changelog [work in progress]

This release starts the 'hardening' cycle for uWSGI 2.0 (scheduled for the end of december 2013).

The metrics subsystem was the last piece missing and this version (after 1 year of analysis) finally includes it.

During the following 2 months we will start deprecating features or plugins that got no-interest, are known to be broken or are simply superseed
by more modern/advanced ones.

Currently the following plugin and features are scheduled for removal:

- The Go plugin, superseeded by the gccgo one. (eventually the Go plugin will be brought back if something changes in the fork() support)
- Auto-snapshotting, was never documented, it has tons of corner case bugs and it is huber-complex. The features added by the :doc:`MasterFifo` allows for better implementations of snapshotting.

Waiting for decision:

- the erlang plugin is extremely old, was badly engineered and should be completely rewritten. If you are a user of it, please contact the staff. Very probably we will not be able to maintain it without sponsorship.
- the matheval support could be removed soon (unless we find some specific use that could require it), substituted by some form of simple math directly implemented in the option parser

Bugfixes
********

- completely skip cgroups initialization when non-root
- tons of post-static_analysis fixes by Riccardo Magliocchetti
- fixed the greenlet plugin reference counting
- avoid kevent storm for stats pusher thread
- fixed rbtimers math

New features
************

The Metrics subsystem
^^^^^^^^^^^^^^^^^^^^^

This was the last piece missing before uWSGI 2.0. The Metrics subsystem allows you to store "numbers" related to monitoring, graphing and quality checks and exports them in various ways.

Official docs: :doc:`Metrics`

The Tornado loop engine
^^^^^^^^^^^^^^^^^^^^^^^

While working on nodejs integration we realized that contrary to what we used to believe, Tornado (an asynchronous, callback based module for python) is usable in uWSGI.

Note: The plugin is not built-in by default

Official docs: :doc:`Tornado`

The 'puwsgi' protocol
^^^^^^^^^^^^^^^^^^^^^

--vassal-set
^^^^^^^^^^^^


Availability
************
