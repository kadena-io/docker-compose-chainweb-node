Usage
=====

Hardware requirements:

* 4 CPU cores,
* 8GB of RAM, and
* 256 GB of disk space.

For supporting higher API loads we recommend to use at least 8 CPU cores and
16GB of RAM.

Networking:

Chainweb node is configured to use port 1789 to communicate with other Chainweb
nodes.

*The P2P port must be reachable from the internet.*

The HTTP REST API of Chainweb node is available on the host on port 1848.

Initialize Database
-------------------

```
docker compose run chainweb-initialize-db
```

The resulting database is *untrusted*. It is fine for use in testing and
non-critical applications.

Validate Database
-----------------

For production applications it is highly recommended to validate the database
after initialization.

```
docker compose run chainweb-validate-db-sync
docker compose run chainweb-validate-db
```

The first command can be skipped if the database has been initialized already.

The second command can take several hours depending on available hardware.
Currently, it takes about 6 hours on a cloud VM with eight CPU cores and eight
GB of RAM. Adding more CPU cores will speed up the process.

Run Chainweb Node
-----------------

Prerequisite: an initialized and possibly validated database.

```
docker compose up -d
```

The service API of the node is available on the docker host at port 1848.

Options
-------

By default the node runs in the Kadena mainnet. To run a node in the Kadena
testnet define the `KADENA_NETWORK` variable in an `.env` file:

```
cat >> .env <<EOF
KADENA_NETWORK=testnet04
```

Database synchronization takes less time when the database is synchronized
from a geographically close location, ideally, in the same data center:

```
cat >> .env <<EOF
DB_SYNC_SERVER=INSERT_IP_ADDRESS_OR_DOMAIN_NAME_OF_NODE
```

If you have already a node running to you can make its database available for
remote synchronization as follows:

```
docker compose up -d chainweb-db-rsync
```

