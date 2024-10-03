# Joining an Existing Testnet

In this section, we use Docker Compose to run a validator and join an existing Testnet.

## One-Click Deploy

> Note: This section was only tested under Linux.

After downloading the `linera-protocol` repository, and checkout the testnet branch `{{#include ../../../TESTNET_BRANCH}}` you can run `scripts/deploy-validator.sh <hostname>` to deploy a Linera validator.

For example:

```bash
$ git fetch origin
$ git checkout -t origin/{{#include ../../../TESTNET_BRANCH}}
$ ./scripts/deploy-validator.sh linera.mydomain.com
```

The public key will be printed after the command has finished executing, for example:

```bash
$ ./scripts/deploy-validator.sh linera.mydomain.com
...
Public Key: 92f934525762a9ed99fcc3e3d3e35a825235dae133f2682b78fe22a742bac196
```

The public key, in this case beginning with `92f`, must be communicated to the Linera Protocol core team along with the chosen host name for onboarding in the next epoch.

For a more bespoke deployment, refer to the manual installation instrucitons below.

## Manual Installation

This section covers everything you need to install to run a Linera validator node with Docker Compose.

> Note: This section was only tested under Linux.

### Docker Compose Requirements

To install Docker Compose see the [installing Docker Compose](https://docs.docker.com/compose/install/) section in the Docker docs.

### Installing the Linera Toolchain

> When installing the Linera Toolchain, you **must** check out the `{{#include ../../../TESTNET_BRANCH}}` branch.

To install the Linera Toolchain refer to the [installation section](../../developer-manual/getting\_started/installation.md#installing-from-github).

You want to install the toolchain from GitHub, as you'll be using the repository to run the Docker Compose validator service.

## Setting up a Linera Validator

For the next section, we'll be working out of the `docker` subdirectory in the `linera-protocol` repository.

### Infrastructure Configuration

Validators run via Docker Compose do not come with a pre-packaged load balancer to perform TLS termination (unlike validators running on Kubernetes).

The load balancer configuration **must** have the following properties:

1. Support HTTP/2 connections.
2. Support gRPC connections.
3. Support long-lived HTTP/2 connections.
4. Support a maximum body size of up to 20 MB.
5. Provide TLS termination with a certificate signed by a known CA.

Finally, the load balancer that performs TLS termination must redirect traffic from `443` to `19100` (the port exposed by the proxy).

### Creating your Validator Configuration

Validators are configured using a TOML file. You can use the following template to set up you own validator configuration:

```toml
server_config_path = "server.json"
host = "<your-host>" # e.g. my-subdomain.my-domain.net
port = 19100
metrics_host = "proxy"
metrics_port = 21100
internal_host = "proxy"
internal_port = 20100
[external_protocol]
Grpc = "ClearText" # Depending on your load balancer you may need "Tls" here.
[internal_protocol]
Grpc = "ClearText"

[[shards]]
host = "shard"
port = 19100
metrics_host = "shard"
metrics_port = 21100

```

### Genesis Configuration

The genesis configuration describes the committee of validators and chains at the point of network creation. It is required for validators to function.

Initially, the genesis configuration for each Testnet will be found in a public bucket managed by the Linera Protocol core team.

An example can be found here:

```bash
wget "https://storage.googleapis.com/linera-io-dev-public/{{#include ../../../TESTNET_DOMAIN}}/genesis.json"
```

### Creating Your Keys

Now that the [validator configuration](joining.md#creating-your-validator-configuration) has been created and the [genesis configuration](joining.md#genesis-configuration) is available, the validator private keys can be generated.

To generate the private keys, the `linera-server` binary is used:

```bash
linera-server generate --validators /path/to/validator/configuration.toml
```

This will generate a file called `server.json` with the information required for a validator to operate, including a cryptographic keypair.

The public key will be printed after the command has finished executing, for example:

```bash
$ linera-server generate --validators /path/to/validator/configuration.toml
2024-07-01T16:51:32.881255Z  INFO linera_version::version_info: Linera protocol: v0.12.0
2024-07-01T16:51:32.881273Z  INFO linera_version::version_info: RPC API hash: p//G+L8e12ZRwUdWoGHWYvWA/03kO0n6gtgKS4D4Q0o
2024-07-01T16:51:32.881274Z  INFO linera_version::version_info: GraphQL API hash: KcS5z1lEg+L9QjcP99l5vNSc7LfCwnwEsfDvMZGJ/PM
2024-07-01T16:51:32.881277Z  INFO linera_version::version_info: WIT API hash: p//G+L8e12ZRwUdWoGHWYvWA/03kO0n6gtgKS4D4Q0o
2024-07-01T16:51:32.881279Z  INFO linera_version::version_info: Source code: https://github.com/linera-io/linera-protocol/tree/44b3e1ab15 (dirty)
2024-07-01T16:51:32.881519Z  INFO linera_server: Wrote server config server.json
92f934525762a9ed99fcc3e3d3e35a825235dae133f2682b78fe22a742bac196 # <- Public Key
```

The public key, in this case beginning with `92f`, must be communicated to the Linera Protocol core team along with the chosen host name for onboarding in the next epoch.

> Note: Before being included in the next epoch, validator nodes will receive no traffic from existing users.

### Building the Linera Docker image

To build the Linera Docker image, run the following command from the root of the `linera-protocol` repository:

```bash
docker build -f docker/Dockerfile . -t linera
```

This can take several minutes.

### Running a Validator Node

Now that the genesis configuration is available at `docker/genesis.json` and the server configuration is available at `docker/server.json`, the validator can be started by running from inside the `docker` directory:

```bash
cd docker && docker compose up -d
```

This will run the Docker Compose deployment in a detached mode. It can take a few minutes for the ScyllaDB image to be downloaded and started.