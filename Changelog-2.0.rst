Changelog-2.0
=============

This is a memo for what we plan to include in uWSGI 2.0 (LTS)

Metric subsystem
****************

think about persistent storage


Better Erlang integration
*************************

remove dependancies with libei

Corerouters backup nodes
************************

On-demand threading mode
************************

Instead of pre-spawning threads in each worker, just spawn a single one that will generate a new one
at each request.

Emperor binary patching
***********************

Emperor clustering
******************

The Legion subsystem will be integrated with the Emperor, allowing the cluster to distribute multiple application on multiple nodes in a balanced way.
Broodlord mode will be updated accordingly.

