# Docker Compose Setup For A Chainweb Mining Node

Before you start familiarize yourself with the Kadena blockchain. You should
have a rough understanding of key pairs, different chains, Pact accounts (in
particular k-accounts), and how to transfer funds between accounts on the same
and on different chains. You should also decide how to store the KDA that you
are going to mine.

You should also know how to use docker and docker compose and understand the
basic concepts of containerized applications and services. Installing docker
and docker compose V2, unfortunately, isn't always trivial on Linux systems.

# Prerequesites

1.  Setup a machine that is directly reachable from the public internet on
    port 1789. If needed, configure your firewall or NAT router accordingly.

    Database initialization will go faster on a more powerful machine. After
    that, for normal node operation, 2 to 4 CPU cores and 8GB of RAM are
    sufficient. Disk size should be at least 120GB, using more will make your
    node future proof.

2.  Install docker and docker client with support for docker compose V2.

    Note that if you run chainweb-node on a cloud VM, it is enough if the docker
    daemon is installed on the cloud VM and docker compose V2 is installed on a
    local client. You can than use the `--host` option of the docker CLI to
    connect and run docker compose commands on the server machine. For instance,
    you could use a container optimized operating system on a cloud VM and use
    docker desktop on your local computer to run docker compose.

# Configure the Node

3.  Create a directory that contains the `docker-compose.yaml` and the `.env`
    file from this repository.

4.  Optionally, edit the `.env` file to match your needs or set the respective
    environment variables. You have to provide your public mining key.

# Pull Container Images

```sh
docker compose pull
```

# Initialize Database Volume

## Download Chain Database

The first step when initializing a Chainweb database is to download an exiting
chain database (stored as RocksDB) to a local docker volume (named
`chainweb-db`).

```sh
docker compose --profile initialize-db up -d chainweb-initialize-db-sync
```

One can use `docker compose logs chainweb-initialize-db-sync` to monitor progress.

The download is complete when `chainweb-initialize-db-sync` exits with code `0`.

One can check the exit code by running

```sh
docker compose ps
```

## Validate Chain Database and Create Pact Databases

Once the download is complete, the database must be validated and the Pact
databases for each chain must be build.

The command `chainweb-initialize-db` performs two separate tasks:

1.  Fully validate the Merkle tree of the chain database (RocksDB). This
    includes scanning and hashing all data and ensuring that the data
    matches the block hashes of the latest cash stored that is stored in the
    database.

    Chain database validation has been successful when the logs contain a log
    message that states

    ```sh
    finished pruning databases
    ```

2.  Rebuild the pact databases (slqite) for all chains so that the Pact state
    for each chain matches the respective block header in latest cut that is
    stored int the chain database (RocksDB).

```sh
docker compose --profile initialize-db up -d chainweb-initialize-db
```

This command will take a long time to complete. Depending on your hardware it
can take between 1.5 and 12 hours. Some cloud providers allow you to add and
remove RAM and CPU cores for an existing VM. It may help to temporarily add
cores (ideally more than 20) during this step. Alternatively, you may attach
the disk to a new VM after initialization is done. In any case, make sure that
your disk provides enough read IOPS to saturate all available cores. If you find
that most CPUs are blocked on IO (CPU cores are idle), try to increase disk read
performance.

One can use `docker compose logs chainweb-initialize-db` to monitor progress.

Database initialization is complete when the following command shows that all
containers exited with code `0`.

```sh
docker compose ps
```

At this point you have a Chainweb database that is fully validated and trusted.
It is stored on a docker volume that is called `chainweb-db`. You should always
keep up-to-date backup copies of this volume, because recreating it would take
a long time.

# Start Chainweb Node and Stratum Server

```sh
docker compose up -d
```

The following command can be used check that the node is healthy.

```sh
docker compose ps
```

By default chainweb-node is configured to expose the node mining API on port
1848. It is strongly recommended to not expose that port publicly. Th stratum
server is exposed on the host on port 1917 that can be used to mining with an
ASIC. Depending on your setup you may have to configure the firewall or NAT so
that the ASIC miner can connect to port 1917 on the machine where the stratum
server is running.

# Remarks

You can speed up the database initialization process by using a powerful machine
with many cores (20-30 cores is ideal) and a larger amount of RAM (16 or more
GB). You want to use a fast disk (SSD or fast HDD). If you use a different
machine for database initialization than for running chainweb-node, you must
manually transfer the docker volume with the Chainweb database to the
chainweb-node machine.

For advanced security it is also possible to modify `docker-compose.yaml` to
restrict the service API port to the docker compose network and not even expose
it on the host).

# Trouble Shooting

It sometimes can happen that errors occur during initial database
synchronization. This results in `chainweb-initialize-db` failing with an error
message during the validation phase (log message are tagged with
`sub-component=headers-checked`). This is resolved by deleting the docker volume
with the database (`chainweb-db`) and start over again.

Chain database validation has been successful when the logs contain a log
message that states

```sh
finished pruning databases
```

If `chainweb-initialize-db` finishes db validation but fails with an non-zero
exit code during Pact database synchronization, it can mean two things:

1.  Some random error happened during the process (e.g. IO error, system running
    out of RAM, the node received an external signal like Ctrl-C, etc.)

2.  Something is wrong with the chainweb-node binary.

If the reason was a random temporary failure the issue can usually be resolved
by repeating the `chainweb-initialize-db` (after eliminating the reason for the
external failure as necessary). When restarting `chainweb-initialize-db` it is
possible to skip full chain database validation if this had already succeeded
before the failure occurred. To skip chain database validation modify the
docker-compose file and change the value of
`services.chainweb-initialize-db-config.command[0].chainweb.cuts.pruneChainDatabase`
from `headers-checked` to `none`. After that `chainweb-initialize-db` will
resume the validation from the point it failed before.

**In any case** never use a Chainweb database in production if validation didn't
complete successfully.

