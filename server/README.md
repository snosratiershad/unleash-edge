# Unleash Edge

[![crates.io](https://img.shields.io/crates/v/unleash-edge?label=latest)](https://crates.io/crates/unleash-edge)
[![Documentation](https://docs.rs/unleash-edge/badge.svg?version=latest)](https://docs.rs/unleash-edge/latest)
![MIT licensed](https://img.shields.io/crates/l/unleash-edge.svg)
[![Dependency Status](https://deps.rs/crate/unleash-edge/12.0.0/status.svg)](https://deps.rs/crate/unleash-edge/12.0.0)
[![CI](https://github.com/Unleash/unleash-edge/actions/workflows/test-with-coverage.yaml/badge.svg)](https://github.com/Unleash/unleash-edge/actions/workflows/test-with-coverage.yaml)
[![Coverage Status](https://coveralls.io/repos/github/Unleash/unleash-edge/badge.svg?branch=main)](https://coveralls.io/github/Unleash/unleash-edge?branch=main)
![downloads](https://img.shields.io/crates/d/unleash-edge.svg)

Unleash Edge is the successor to the [Unleash Proxy](https://docs.getunleash.io/how-to/how-to-run-the-unleash-proxy).

Unleash Edge sits between the Unleash API and your SDKs and provides a cached read-replica of your Unleash instance. This means you can scale up your Unleash instance to thousands of connected SDKs without increasing the number of requests you make to your Unleash instance.

Unleash Edge offers two important features:

- **Performance**: Unleash Edge caches in memory and can run close to your end-users. A single instance can handle tens to hundreds of thousands of requests per second.
- **Resilience**: Unleash Edge is designed to survive restarts and operate properly even if you lose connection to your Unleash server.

Unleash Edge is built to help you scale Unleash, if you're looking for the easiest way to connect your client SDKs you can check out our [Frontend API](https://docs.getunleash.io/reference/front-end-api).

## Migrating to Edge from the Proxy

For more info on migrating, check out the [migration guide](./migration-guide.md) that details the differences between Edge and the Proxy and how to achieve similar behavior in Edge.

## Running Unleash Edge

Unleash Edge is compiled to a single binary. You can configure it by passing in arguments or setting environment variables.

```shell
Usage: unleash-edge [OPTIONS] <COMMAND>

Commands:
  edge     Run in edge mode
  offline  Run in offline mode
  help     Print this message or the help of the given subcommand(s)

Options:
  -p, --port <PORT>
          Which port should this server listen for HTTP traffic on [env: PORT=] [default: 3063]
  -i, --interface <INTERFACE>
          Which interfaces should this server listen for HTTP traffic on [env: INTERFACE=] [default: 0.0.0.0]
  -b, --base-path <BASE_PATH>
          Which base path should this server listen for HTTP traffic on [env: BASE_PATH=] [default: ]
  -w, --workers <WORKERS>
          How many workers should be started to handle requests. Defaults to number of physical cpus [env: WORKERS=] [default: number of physical cpus]
      --tls-enable
          Should we bind TLS [env: TLS_ENABLE=]
      --tls-server-key <TLS_SERVER_KEY>
          Server key to use for TLS [env: TLS_SERVER_KEY=] (Needs to be a path to a file)
      --tls-server-cert <TLS_SERVER_CERT>
          Server Cert to use for TLS [env: TLS_SERVER_CERT=] (Needs to be a path to a file)
      --tls-server-port <TLS_SERVER_PORT>
          Port to listen for https connection on (will use the interfaces already defined) [env: TLS_SERVER_PORT=] [default: 3043]
      --instance-id <INSTANCE_ID>
          Instance id. Used for metrics reporting [env: INSTANCE_ID=] [default: Ulid::new()]
  -a, --app-name <APP_NAME>
          App name. Used for metrics reporting [env: APP_NAME=] [default: unleash-edge]
  -h, --help
          Print help
```

### Built-in Health check
There is now (from 5.1.0) a subcommand named `health` which will ping your health endpoint and exit with status 0 provided the health endpoint returns 200 OK.

Example:
```shell
./unleash-edge health
```
will check an Edge process running on http://localhost:3063. If you're using base-path or the port variable you should use the `-e --edge-url` CLI arg (or the EDGE_URL environment variable) to tell the health checker where edge is running.

If you're hosting Edge with a self-signed certificate using the tls cli arguments, you should use the `--ca-certificate-file <file_containing_your_ca_and_key_in_pem_format>` flag (or the CA_CERTIFICATE_FILE environment variable) to allow the health checker to trust the self signed certificate.

### Built-in Ready check
There is now (from 12.0.0) a subcommand named `ready` which will ping your ready endpoint and exit with status 0 provided the ready endpoint returns 200 OK and `{ status: "READY" }`. Otherwise it will return status 1 and an error message to signal that Edge is not ready (it has not spoken to upstream or recovered from a persisted backup).

Examples:
* Edge not running:
```shell
$ ./unleash-edge ready
Error: Failed to connect to ready endpoint at http://localhost:3063/internal-backstage/ready. Failed with status None
$ echo $?
1
```
 
* Edge running but not populated its feature cache yet (not spoken to upstream or restored from backup)
```shell
$ ./unleash-edge ready
Error: Ready check returned a different status than READY. It returned EdgeStatus { status: NotReady }
$ echo $?
1
```
* Edge running and synchronized. I.e. READY
```shell
$ ./unleash-edge ready
OK
$ echo $?
0
```

If you're hosting Edge with a self-signed certificate using the tls cli arguments, you should use the `--ca-certificate-file <file_containing_your_ca_and_key_in_pem_format>` flag (or the CA_CERTIFICATE_FILE environment variable) to allow the health checker to trust the self signed certificate.


## Getting Unleash Edge

Unleash Edge is distributed as a binary and as a docker image.

### Binary

- The binary is downloadable from our [Releases page](https://github.com/Unleash/unleash-edge/releases/latest).
- We're currently building for linux x86_64, windows x86_64, darwin (OS X) x86_64 and darwin (OS X) aarch64 (M1/M2 macs)

### Docker

- The docker image gets uploaded to dockerhub and Github Package registry.
- For dockerhub use the coordinates `unleashorg/unleash-edge:<version>`.
- For Github package registry use the coordinates `ghpr.io/unleash/unleash-edge:<version>`
- If you'd like to live on the edge (sic) you can use the tag `edge`. This is built from `HEAD` on each commit
- When running the docker image, the same CLI arguments that's available when running the binary is available to your `docker run` command. To start successfully you will need to decide which mode you're running in.
  - If running in `edge` mode your command should be
    - `docker run -p 3063:3063 -e UPSTREAM_URL=<YOUR_UNLEASH_INSTANCE> unleashorg/unleash-edge:v8.0.1 edge`
  - If running in `offline` mode you will need to provide a volume containing your feature toggles file. An example is available inside the examples folder. To use this, you can use the command
    - `docker run -v ./examples:/edge/data -p 3063:3063 -e BOOTSTRAP_FILE=/edge/data/features.json -e TOKENS='my-secret-123,another-secret-789' unleashorg/unleash-edge:v8.0.1 offline`

### Cargo/Rust

If you have the [Rust toolchain](https://rustup.rs) installed you can build a binary for the platform you're running by cloning this repo and running `cargo build --release`. This will give you an `unleash-edge` binary in `./target/release`

## Concepts

### Modes

Edge currently supports 2 different modes:

- [Edge](#edge) - Connection to upstream node (Unleash instance or another Edge). Supports dynamic tokens, metrics and other advanced features;
- [Offline](#offline) - No connection to upstream node. Full control of data and tokens;

#### Edge

```mermaid
graph LR
  A(Client) -->|Fetch toggles| B((Edge))
  B-->|Fetch toggles| C((Unleash))
```

Edge mode is the "standard" mode for Unleash Edge and the one you should default to in most cases. It connects to an upstream node, such as your Unleash instance, and uses that as the source of truth for feature toggles.

Other than connecting Edge directly to your Unleash instance, it's also possible to connect to another Edge instance (_daisy chaining_). You can have as many Edge nodes as you'd like between the Edge node your clients are accessing and the Unleash server, and it's also possible for multiple nodes to connect to a single upstream one. Depending on your architecture and requirements this can be a powerful feature, offering you flexibility and scalability when planning your implementation.

```mermaid
graph LR
  A(Client 1) -->|Fetch toggles| C((Edge 1))
  B(Client 2) -->|Fetch toggles| D((Edge 2))
  C-->|Fetch toggles| E((Edge 3))
  D-->|Fetch toggles| E
  E-->|Fetch toggles| F((Unleash))
```

This means that, in order to start up, Edge mode needs to know where the upstream node is. This is done by passing the `--upstream-url` command line argument or setting the `UPSTREAM_URL` environment variable.

By default, Edge mode uses an in-memory cache to store the features it fetches from the upstream node. However, you may want to use a more persistent storage solution. For this purpose, Edge supports either Redis or a backup file, which you can configure by passing in either the `--redis-url` or `--backup_folder` command line argument, respectively. On start-up, Edge checks whether the persistent backup option is specified, in which case it uses it to populate its internal caches. This can be useful when your Unleash server is unreachable.

Edge mode also supports dynamic tokens, meaning that Edge doesn't need a token to be provided when starting up. Once we make a request to the `/api/client/features` endpoint using a [client token](https://docs.getunleash.io/reference/api-tokens-and-client-keys#client-tokens) Edge will validate upstream and fetch its respective features. After that, it gets added to the list of known tokens that gets periodically synced, making sure it is a valid token and its features are up-to-date.

Even though Edge supports dynamic tokens, you still have the option of providing a token through the command line argument or environment variable. This way, since Edge already knows about your token at start up, it will sync your features for that token and should be ready for your requests right away (_warm up / hot start_).

### Front-end tokens
[Front-end tokens](https://docs.getunleash.io/reference/api-tokens-and-client-keys#front-end-tokens) can also be used with `/api/frontend` and `/api/proxy` endpoints, however they are not allowed to fetch features upstream.
In order to use these tokens correctly and make sure they return the correct information, it's important that the features they are allowed to access are already present in that Edge node's features cache.
The easiest way to ensure this is by passing in at least one client token as one of the command line arguments,
ensuring it has access to the same features as the front-end token you'll be using.
If you're using a frontend token that doesn't have data in the node's feature cache, you will receive an HTTP Status code: 511 Network Authentication Required along with a body of which project and environment you will need to add a client token for.

#### Enterprise
Using `--service-account-token` CLI arg or `SERVICE_ACCOUNT_TOKEN` environment variable you can provide Edge with a [Service Account Token](https://docs.getunleash.io/reference/service-accounts) which has access to create client tokens at startup.
Doing so, Edge will use this token to create a client token for any frontend token where Edge is not already aware of a client token which will give it access to the necessary projects for creating the response.

#### Open Source
Unleash OSS does not support Service accounts, so if you want Edge to create Client tokens for Frontend tokens you will need to use an admin token in the `--service-account-token | SERVICE_ACCOUNT_TOKEN` argument.


```json
{
  "access": {
    "environment": "default",
    "project": "demo-app"
  },
  "explanation": "Edge does not yet have data for this token. Please make a call against /api/client/features with a client token that has the same access as your token"
}
```

To launch in this mode, run:

```bash
$ unleash-edge edge -h
Run in edge mode

Usage: unleash-edge edge [OPTIONS] --upstream-url <UPSTREAM_URL>

Options:
  -u, --upstream-url <UPSTREAM_URL>
          Where is your upstream URL. Remember, this is the URL to your instance, without any trailing /api suffix [env: UPSTREAM_URL=]
  -r, --redis-url <REDIS_URL>
          A URL pointing to a running Redis instance. Edge will use this instance to persist feature and token data and read this back after restart. Mutually exclusive with the --backup-folder option [env: REDIS_URL=]
  -b, --backup-folder <BACKUP_FOLDER>
          A path to a local folder. Edge will write feature and token data to disk in this folder and read this back after restart. Mutually exclusive with the --redis-url option [env: BACKUP_FOLDER=]
  -m, --metrics-interval-seconds <METRICS_INTERVAL_SECONDS>
          How often should we post metrics upstream? [env: METRICS_INTERVAL_SECONDS=] [default: 60]
  -f, --features-refresh-interval-seconds <FEATURES_REFRESH_INTERVAL_SECONDS>
          How long between each refresh for a token [env: FEATURES_REFRESH_INTERVAL_SECONDS=] [default: 10]
      --token-revalidation-interval-seconds <TOKEN_REVALIDATION_INTERVAL_SECONDS>
          How long between each revalidation of a token [env: TOKEN_REVALIDATION_INTERVAL_SECONDS=] [default: 3600]
  -t, --tokens <TOKENS>
          Get data for these client tokens at startup. Accepts comma-separated list of tokens. Hot starts your feature cache [env: TOKENS=]
  -H, --custom-client-headers <CUSTOM_CLIENT_HEADERS>
          Expects curl header format (-H <HEADERNAME>: <HEADERVALUE>) for instance `-H X-Api-Key: mysecretapikey` [env: CUSTOM_CLIENT_HEADERS=]
  -s, --skip-ssl-verification
          If set to true, we will skip SSL verification when connecting to the upstream Unleash server [env: SKIP_SSL_VERIFICATION=]
      --pkcs8-client-certificate-file <PKCS8_CLIENT_CERTIFICATE_FILE>
          Client certificate chain in PEM encoded X509 format with the leaf certificate first. The certificate chain should contain any intermediate certificates that should be sent to clients to allow them to build a chain to a trusted root [env: PKCS8_CLIENT_CERTIFICATE_FILE=]
      --pkcs8-client-key-file <PKCS8_CLIENT_KEY_FILE>
          Client key is a PEM encoded PKCS#8 formatted private key for the leaf certificate [env: PKCS8_CLIENT_KEY_FILE=]
      --pkcs12-identity-file <PKCS12_IDENTITY_FILE>
          Identity file in pkcs12 format. Typically this file has a pfx extension [env: PKCS12_IDENTITY_FILE=]
      --pkcs12-passphrase <PKCS12_PASSPHRASE>
          Passphrase used to unlock the pkcs12 file [env: PKCS12_PASSPHRASE=]
      --upstream-certificate-file <UPSTREAM_CERTIFICATE_FILE>
          Extra certificate passed to the client for building its trust chain. Needs to be in PEM format (crt or pem extensions usually are) [env: UPSTREAM_CERTIFICATE_FILE=]

  -h, --help
          Print help

```

#### Offline

```mermaid
graph LR
  A(Client) -->|Fetch toggles| B((Edge))
  B-->|Fetch toggles| C[Features dump]
```

Offline mode is useful when there is no connection to an upstream node, such as your Unleash instance or another Edge instance, or as a tool to make working with Unleash easier during development.

To use offline mode, you'll need a features file. The easiest way to get one is to download a JSON dump of a result from a query against an Unleash server on the [/api/client/features](https://docs.getunleash.io/reference/api/unleash/get-client-feature) endpoint. You can also use a hand rolled, human readable JSON version of the features file. Edge will automatically convert it to the API format when it starts up. Here's an example:

``` json
{
  "featureOne": {
    "enabled": true,
    "variant": "variantOne"
  },
  "featureTwo": {
    "enabled": false,
    "variant": "variantTwo"
  },
  "featureThree": {
    "enabled": true
  }
}
```

The simplified JSON format should be an object with a key for each feature. You can force the result of `is_enabled` in your SDK by setting the enabled property, likewise can also force the result of `get_variant` by specifying the name of the variant you want. This format is primarily for development.

When using offline mode you must specify one or more tokens at startup. These tokens will let your SDKs access Edge. Tokens following the Unleash API format `[project]:[environment].<somesecret>` allow Edge to recognize the project and environment specified in the token, returning only the relevant features to the calling SDK. On the other hand, for tokens not adhering to this format, Edge will return all features if there is an exact match with any of the startup tokens.

To make local development easier, you can specify a reload interval in seconds (Since Unleash-Edge 10.0.x); this will cause Edge to reload the features file from disk every X seconds. This can be useful for local development.

Since offline mode does not connect to an upstream node, it does not support metrics or dynamic tokens.

To launch in this mode, run:

```bash
$ ./unleash-edge offline --help
Usage: unleash-edge offline [OPTIONS]

Options:
  -b, --bootstrap-file <BOOTSTRAP_FILE>         [env: BOOTSTRAP_FILE=]
  -t, --tokens <TOKENS>                         [env: TOKENS=]
  -r, --reload-interval <RELOAD_INTERVAL>       [env: RELOAD_INTERVAL=]

```

##### Environments in offline mode
Currently, Edge does not support multiple environments in offline mode. All tokens added at startup will receive the same list of features passed in as the bootstrap argument. 
However, tokens in <project>:<environment>.<secret> format will still filter by project.

## [Metrics](https://docs.getunleash.io/reference/api/unleash/metrics)

**❗ Note:** For Unleash to correctly register SDK usage metrics sent from Edge instances, your Unleash instance must be v4.22 or newer.

Since Edge is designed to avoid overloading its upstream, Edge gathers and accumulates usage metrics from SDKs for a set interval (METRICS_INTERVAL_SECONDS) before posting a batch upstream.
This reduces load on Unleash instances down to a single call every interval, instead of every single client posting to Unleash for updating metrics.
Unleash instances running on versions older than 4.22 are not able to handle the batch format posted by Edge, which means you won't see any metrics from clients connected to an Edge instance until you're able to update to 4.22 or newer.

## Performance

Unleash Edge will scale linearly with CPU. There are k6 benchmarks in the benchmark folder. We've already got some initial numbers from [hey](https://github.com/rakyll/hey).

Do note that the number of requests Edge can handle does depend on the total size of your toggle response. That is, Edge is faster if you only have 10 toggles with 1 strategy each, than it will be with 1000 toggles with multiple strategies on each. Benchmarks here were run with data fetched from the Unleash demo instance (roughly 100kB (350 features / 200 strategies)) as well as against a small dataset of 5 features with one strategy on each.

Edge was started using
`docker run --cpus="<cpu>" --memory=128M -p 3063:3063 -e UPSTREAM_URL=<upstream> -e TOKENS="<client token>" unleashorg/unleash-edge:edge -w <number of cpus rounded up to closest integer> edge`

Then we run hey against the proxy endpoint, evaluating toggles

### Large Dataset (350 features (100kB))

```shell
$ hey -z 10s -H "Authorization: <frontend token>" http://localhost:3063/api/frontend`
```

| CPU | Memory | RPS   | Endpoint      | p95   | Data transferred |
| --- | ------ | ----- | ------------- | ----- | ---------------- |
| 0.1 | 6.7 Mi | 600   | /api/frontend | 103ms | 76Mi             |
| 1   | 6.7 Mi | 6900  | /api/frontend | 7.4ms | 866Mi            |
| 4   | 9.5    | 25300 | /api/frontend | 2.4ms | 3.2Gi            |
| 8   | 15     | 40921 | /api/frontend | 1.6ms | 5.26Gi           |

and against our client features endpoint.

```shell
$ hey -z 10s -H "Authorization: <client token>" http://localhost:3063/api/client/features
```

| CPU | Memory observed | RPS   | Endpoint             | p95   | Data transferred |
| --- | --------------- | ----- | -------------------- | ----- | ---------------- |
| 0.1 | 11 Mi           | 309   | /api/client/features | 199ms | 300 Mi           |
| 1   | 11 Mi           | 3236  | /api/client/features | 16ms  | 3 Gi             |
| 4   | 11 Mi           | 12815 | /api/client/features | 4.5ms | 14 Gi            |
| 8   | 17 Mi           | 23207 | /api/client/features | 2.7ms | 26 Gi            |

### Small Dataset (5 features (2kB))

```shell
$ hey -z 10s -H "Authorization: <frontend token>" http://localhost:3063/api/frontend`
```

| CPU | Memory  | RPS    | Endpoint      | p95   | Data transferred |
| --- | ------- | ------ | ------------- | ----- | ---------------- |
| 0.1 | 4.3 Mi  | 3673   | /api/frontend | 93ms  | 9Mi              |
| 1   | 6.7 Mi  | 39000  | /api/frontend | 1.6ms | 80Mi             |
| 4   | 6.9 Mi  | 100020 | /api/frontend | 600μs | 252Mi            |
| 8   | 12.5 Mi | 141090 | /api/frontend | 600μs | 324Mi            |

and against our client features endpoint.

```shell
$ hey -z 10s -H "Authorization: <client token>" http://localhost:3063/api/client/features
```

| CPU | Memory observed | RPS    | Endpoint             | p95   | Data transferred |
| --- | --------------- | ------ | -------------------- | ----- | ---------------- |
| 0.1 | 4 Mi            | 3298   | /api/client/features | 92ms  | 64 Mi            |
| 1   | 4 Mi            | 32360  | /api/client/features | 2ms   | 527Mi            |
| 4   | 11 Mi           | 95838  | /api/client/features | 600μs | 2.13 Gi          |
| 8   | 17 Mi           | 129381 | /api/client/features | 490μs | 2.87 Gi          |

## Why choose Unleash Edge over the Unleash Proxy?

Edge offers a superset of the same feature set as the Unleash Proxy and we've made sure it offers the same security and privacy features.

However, there are a few notable differences between the Unleash Proxy and Unleash Edge:

- Unleash Edge is built to be light and fast, it handles an order of magnitude more requests per second than the Unleash Proxy can, while using two orders of magnitude less memory.
- All your Unleash environments can be handled by a single instance, no more running multiple instances of the Unleash Proxy to handle both your development and production environments.
- Backend SDKs can connect to Unleash Edge without turning on experimental feature flags.
- Unleash Edge is smart enough to dynamically resolve the tokens you use to connect to it against the upstream Unleash instance. This means you don't have to worry about knowing in advance what tokens your SDKs use - if you want to swap out the Unleash token your SDK uses, this can be done without ever restarting or worrying about Unleash Edge. Unleash Edge will only collect and cache data for the environments and projects you use.

## Development

See our [Contributors guide](./CONTRIBUTING.md) as well as our [development-guide](./development-guide.md)
