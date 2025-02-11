# Tansu demo on Fly.io with Tigris.io S3 Compatible Storage

[Tansu](https://tansu.io/) is an Apache Kafka&reg; compatible
streaming platform with JSON/Protobuf schema validation,
supporting PostgreSQL, S3 or memory storage engines.
In this demo we will deploy Tansu on [fly](https://fly.io) using their
[integrated S3 service from Tigris](https://fly.io/docs/tigris/).

[Tansu](https://tansu.io/) is a *stateless* broker, meaning
that the broker *can* be shutdown between API requests. We use
[flycast](https://fly.io/docs/networking/flycast/) to spin up (and down)
brokers *automatically*. Automatically scaling to zero: No more idle brokers!

Firstly, download and install the [fly](https://fly.io)
command line with [these instructions](https://fly.io/docs/flyctl/install/).

Create a new directory called `fly-tigris-demo` and use `fly launch`
cloning the `tansu-io/fly-tigris-demo` template:

```shell
mkdir fly-tigris-demo
cd fly-tigris-demo
fly launch --from https://github.com/tansu-io/fly-tigris-demo --no-deploy
```

When asked `? Would you like to copy its configuration to the new app?`, hit `Y`.
When asked `? Do you want to tweak these settings before proceeding?`, hit `N`.

Output will look be something like:

```shell
Launching from git repo https://github.com/tansu-io/fly-tigris-demo
Cloning into '.'...
remote: Enumerating objects: 4, done.
remote: Counting objects: 100% (4/4), done.
remote: Compressing objects: 100% (4/4), done.
remote: Total 4 (delta 0), reused 4 (delta 0), pack-reused 0 (from 0)
Receiving objects: 100% (4/4), done.
An existing fly.toml file was found for app tansu
? Would you like to copy its configuration to the new app? Yes
Using build strategies '[the "ghcr.io/tansu-io/tansu:pr-165" docker image]'. Remove [build] from fly.toml to force a rescan
Creating app in /Users/bickle/tmp/fly-tigris-demo
We're about to launch your app on Fly.io. Here's what you're getting:

Organization: Travis Bickle            (fly launch defaults to the personal org)
Name:         tansu                    (from your fly.toml)
Region:       London, United Kingdom   (this is the fastest region for you)
App Machines: shared-cpu-1x, 256MB RAM (from your fly.toml)
Postgres:     <none>                   (not requested)
Redis:        <none>                   (not requested)
Tigris:       <none>                   (not requested)

? Do you want to tweak these settings before proceeding? No
Created app 'tansu' in organization 'personal'
Admin URL: https://fly.io/apps/tansu
Hostname: tansu.fly.dev
Wrote config file fly.toml
Validating /Users/bickle/tmp/fly-tigris-demo/fly.toml
✓ Configuration is valid
Your app is ready! Deploy with `flyctl deploy`
```

Tansu can use any S3 compatible or PostgreSQL database for storage.
In this demo we will use [Tigris](https://fly.io/docs/tigris/) as it
is integrated with [fly](https://fly.io). Create a new S3 bucket
on Tigris using:

```shell
fly storage create
```

Picking a name of your choice, or using the default:

```shell
? Choose a name, use the default, or leave blank to generate one: tansu
Your Tigris project (tansu) is ready. See details and next steps with: https://fly.io/docs/reference/tigris/

Setting the following secrets on tansu:
AWS_ACCESS_KEY_ID: tid_YOUR_ACCESS_KEY_ID
AWS_ENDPOINT_URL_S3: https://fly.storage.tigris.dev
AWS_REGION: auto
AWS_SECRET_ACCESS_KEY: tsec_YOUR_SECRET_KEY_ID
BUCKET_NAME: tansu

Secrets are staged for the first deployment
```

Some secrets have been created, which you can verify with

```shell
fly secrets list
```

Our [fly.toml](https://github.com/tansu-io/fly-tigris-demo/blob/c01bef2a4bb18a153cf37aa1e81c0f3157fb9bd2/fly.toml#L14) is
already setup to use these secrets when communicating to Tigris S3:

```shell
NAME                    DIGEST                  CREATED AT 
AWS_ACCESS_KEY_ID       8cbc2a5f5f704ae7        1m12s ago       
AWS_ENDPOINT_URL_S3     85e8ac62d7de0c23        1m12s ago       
AWS_REGION              274e16452b90854d        1m12s ago       
AWS_SECRET_ACCESS_KEY   08469f68dfc80810        1m12s ago       
BUCKET_NAME             057611985ecb3ae3        1m12s ago       
```

Allocate a [flycast](https://fly.io/docs/networking/flycast/) private IPv6 address
to connect with the Tansu brokers:

```
fly ips allocate-v6 --private
```

Finally, deploy the application onto [fly](https://fly.io) with:

```shell
fly deploy
```

Start an interactive shell using the Apache Kafka&reg; Java client to connect to Tansu:

```shell
fly machine run --shell apache/kafka:3.9.0
```

Create a test topic, note that our `bootstrap-server` is `${FLY_APP_NAME}.flycast:9092` using the
[flycast](https://fly.io/docs/networking/flycast/) address allocated earlier:

```
/opt/kafka/bin/kafka-topics.sh \
  --bootstrap-server ${FLY_APP_NAME}.flycast:9092 \
  --partitions=3 \
  --replication-factor=1 \
  --create \
  --topic test
```

Produce a message:

```
echo "hello world" | \
/opt/kafka/bin/kafka-console-producer.sh \
  --bootstrap-server ${FLY_APP_NAME}.flycast:9092 \
  --topic test
```

Fetch a message, using a consumer group, there is a short delay while the group is formed:

```shell
/opt/kafka/bin/kafka-console-consumer.sh \
  --bootstrap-server ${FLY_APP_NAME}.flycast:9092 \
  --group test-consumer-group \
  --topic test
```

If you wait a minute or so, the [Tansu](https://tansu.io/) brokers will shut down automatically: scaling to zero.
The brokers are *stateless*, without the overhead of distributed consensus. All data is persisted in
S3 using [optimistic concurrency control](https://en.wikipedia.org/wiki/Optimistic_concurrency_control),
made possible with [conditional write](https://www.tigrisdata.com/blog/s3-conditional-writes/) support,
offered by most S3 vendors.

You can verify whether the brokers are still running using:

```shell
fly machines ls
``` 

Checking that they're all `stopped`. If you now run:

```shell
/opt/kafka/bin/kafka-topics.sh --bootstrap-server ${FLY_APP_NAME}.flycast:9092 --list
```

A broker will automatically restart to handle this request scaling back up from zero.
There is a short delay while waiting
for the broker to become ready. In environments where such a delay isn't acceptable, scaling
to one may be appropriate (using `fly scale`). Brokers can also run in multiple regions without
overhead because they are *stateless*.