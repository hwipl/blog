# Private IPv4 Addressing Example

## Example Network

The example network is shown in the following figure:

```
                    ________
                   /        \
                  /   Other  \
                  \ Networks /
                   \________/
                       |
.......................|.......................
: Network              |                      :
:     __________       |       __________     :
:    /          \      |      /          \    :
:   /   Site 1   \     |     /   Site 2   \   :
:  /              \    |    /              \  :
:  |  +--------+  |    |    |  +--------+  |  :
:  |  | Node 1 |  |    |    |  | Node 1 |  |  :
:  |  +--------+  |    |    |  +--------+  |  :
:  |              |    |    |              |  :
:  |  +--------+  |    |    |  +--------+  |  :
:  |  | Node 2 |  |____|____|  | Node 2 |  |  :
:  |  +--------+  |         |  +--------+  |  :
:  |              |         |              |  :
:  |  +--------+  |         |  +--------+  |  :
:  |  | Node 3 |  |         |  | Node 3 |  |  :
:  \  +--------+  /         \  +--------+  /  :
:   \            /           \            /   :
:    \__________/             \__________/    :
:.............................................:
```

The network consists of the two sites `Site 1` and `Site 2`. Each site contains
the three nodes `Node 1`, `Node 2` and `Node 3`. All nodes within a site can
reach each-other. Both sites are interconnected. Also, both sites are connected
to other external networks. So, a node in one site can reach all nodes in the
other site as well as nodes in the other networks.

## IPv4 Addressing

The IP addressing is shown in the following table:

| Entity   | IP           |
|----------|--------------|
| Network  | 10.20.0.0/16 |
| Site 1   | 10.20.1.0/24 |
| - Node 1 | 10.20.1.1    |
| - Node 2 | 10.20.1.2    |
| - Node 3 | 10.20.1.3    |
| Site 2   | 10.20.2.0/24 |
| - Node 1 | 10.20.2.1    |
| - Node 2 | 10.20.2.2    |
| - Node 3 | 10.20.2.3    |

The network uses IP addresses in the range `10.20.0.0/16`. Site 1 uses the
subnet `10.20.1.0/24`, Site 2 uses `10.20.2.0/24`. In each site, each node uses
its number as host part of its address.
