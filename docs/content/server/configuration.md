# Configuration

Centrifugo expects JSON, TOML or YAML as configuration file format. Thanks to brilliant Go library for application configuration - [viper](https://github.com/spf13/viper).

First let's look at all available command-line options:

```bash
centrifugo -h
```

You should see something like this as output:

```
Centrifugo – real-time messaging server

Usage:
   [flags]
   [command]

Available Commands:
  checkconfig Check configuration file
  genconfig   Generate simple configuration file to start with
  help        Help about any command
  version     Centrifugo version information

Flags:
  -a, --address string             interface address to listen on
      --admin                      enable admin web interface
      --admin_insecure             use insecure admin mode – no auth required for admin socket
      --api_insecure               use insecure API mode
      --client_insecure            start in insecure client mode
  -c, --config string              path to config file (default "config.json")
      --debug                      enable debug endpoints
  -e, --engine string              engine to use: memory or redis (default "memory")
      --grpc_api                   enable GRPC API server
  -h, --help                       help for this command
      --internal_port string       custom port for internal endpoints
      --log_file string            optional log file - if not specified logs go to STDOUT
      --log_level string           set the log level: debug, info, error, fatal or none (default "info")
  -n, --name string                unique node name
      --pid_file string            optional path to create PID file
  -p, --port string                port to bind HTTP server to (default "8000")
      --prometheus                 enable Prometheus metrics endpoint
      --redis_db int               Redis database (Redis engine)
      --redis_host string          Redis host (Redis engine) (default "127.0.0.1")
      --redis_master_name string   name of Redis master Sentinel monitors (Redis engine)
      --redis_password string      Redis auth password (Redis engine)
      --redis_port string          Redis port (Redis engine) (default "6379")
      --redis_sentinels string     comma-separated list of Sentinel addresses (Redis engine)
      --redis_tls                  enable Redis TLS connection
      --redis_tls_skip_verify      disable Redis TLS host verification
      --redis_url string           Redis connection URL in format redis://:password@hostname:port/db (Redis engine)
      --tls                        enable TLS, requires an X509 certificate and a key file
      --tls_cert string            path to an X509 certificate file
      --tls_key string             path to an X509 certificate key
```

Keep in mind that all command-line options of Centrifugo can be set via configuration file with the same name (without `--` prefix of course). 

### version

To show version and exit run:

```
centrifugo version
```

### JSON file

Centrifugo requires configuration file on start. As was mentioned earlier it must be a file with valid JSON.

This is a minimal Centrifugo configuration file:

```javascript
{
  "secret": "<YOUR-SECRET-STRING-HERE>",
  "api_key": "<YOUR-API-KEY-HERE>"
}
```

The only two fields required are **secret** and **api_key**. Secret used to check JWT signature (more about JWT in [authentication chapter](authentication.md)). API key used for Centrifugo API endpoint authorization, see more in [chapter about server HTTP API](http_api.md). Keep both values in secret and never reveal to clients.

### TOML file

Centrifugo also supports TOML format for configuration file:

```
centrifugo --config=config.toml
```

Where `config.toml` contains:

```
secret = "<YOUR-SECRET-STRING-HERE>"
api_key = "<YOUR-API-KEY-HERE>"
log_level = "debug"
```

I.e. the same configuration as JSON file above with one extra option.

### YAML file

And YAML config also supported. `config.yaml`:

```
secret: "<YOUR-SECRET-STRING-HERE>"
api_key: "<YOUR-API-KEY-HERE>"
log_level: debug
```

With YAML remember to use spaces, not tabs when writing configuration file.

### checkconfig command

Centrifugo has special command to check configuration file `checkconfig`:

```bash
centrifugo checkconfig --config=config.json
```

If any errors found during validation – program will exit with error message and exit code 1.

### genconfig command

Another command is `genconfig`:

```
centrifugo genconfig -c config.json
```

It will automatically generate the minimal required configuration file.

### Important options

Some of the most important options you can configure when running Centrifugo:

* `address` – bind your Centrifugo to specific interface address (by default `""`)
* `port` – port to bind Centrifugo to (by default `8000`)
* `engine` – engine to use - `memory` or `redis` (by default `memory`). Read more about engines in [special chapter](engines).

Note that some options can be set via command-line. Command-line options are more valuable when set than configuration file's options. See description of [viper](https://github.com/spf13/viper) – to see more details about configuration options priority.

### Channel options

Let's look at options related to channels. Channel is an entity to which clients can subscribe to receive messages published into that channel. Channel is just a string (several symbols has special meaning in Centrifugo - see [special chapter](channels.md) to find more information about channels). The following options will affect channel behaviour:

* `publish` – allow clients to publish messages into channels directly (from client side). Your application will never receive those messages. In idiomatic case all messages must be published to Centrifugo by your application backend using Centrifugo API. But this option can be useful when you want to build something without backend-side validation and saving into database. This option can also be useful for demos and prototyping real-time ideas. By default it's `false`.

* `subscribe_to_publish` - when `publish` option enabled client can publish into channel without being subscribed to it. This option enables automatic check that client subscribed on channel before allowing client to publish into channel.

* `anonymous` – this option enables anonymous access (with empty `sub` claim in connection token). In most situations your application works with authenticated users so every user has its own unique id. But if you provide real-time features for public access you may need unauthorized access to some channels. Turn on this option and use empty string as user ID. By default `false`.

* `presence` – enable/disable presence information. Presence is an information about clients currently subscribed on channel. By default `false` – i.e. no presence information will be available for channels.

* `join_leave` – enable/disable sending join(leave) messages when client subscribes on channel (unsubscribes from channel). By default `false`.

* `history_size` – history size (amount of messages) for channels. As Centrifugo keeps all history messages in memory it's very important to limit maximum amount of messages in channel history to reasonable value. `history_size` defines maximum amount of messages that Centrifugo will keep for **each** channel in namespace during history lifetime (see below). By default history size is `0` - this means that channels will have no history messages at all.

* `history_lifetime` – interval in seconds how long to keep channel history messages. As all history is storing in memory it is also very important to get rid of old history data for unused (inactive for a long time) channels. By default history lifetime is `0` – this means that channels will have no history messages at all. **So to turn on keeping history messages you should wisely configure both `history_size` and `history_lifetime` options**.

* `history_recover` – boolean option, when enabled Centrifugo will try to recover missed publications while client was disconnected for some reason (bad internet connection for example). By default `false`. This option must be used in conjunction with reasonably configured message history for channel i.e. `history_size` and `history_lifetime` **must be set** (because Centrifugo uses channel history to recover messages). Also note that not all real-time events require this feature turned on so think wisely when you need this. When this option turned on your application should be designed in a way to tolerate duplicate messages coming from channel (currently Centrifugo returns recovered publications in order and without duplicates but this is implementation detail that can be theoretically changed in future). See more details about how recovery works in [special chapter](recover.md).

Let's look how to set some of these options in config:

```javascript
{
    "secret": "my-secret-key",
    "api_key": "secret-api-key",
    "anonymous": true,
    "publish": true,
    "subscribe_to_publish": true,
    "presence": true,
    "join_leave": true,
    "history_size": 10,
    "history_lifetime": 300,
    "history_recover": true
}
```

And the last channel specific option is `namespaces`. `namespaces` are optional and if set must be an array of namespace objects. Namespace allows to configure custom options for channels starting with namespace name. This provides a great control over channel behaviour.

Namespace has a name and the same channel options (with same defaults) as described above.

* `name` - unique namespace name (name must consist of letters, numbers, underscores or hyphens and be more than 2 symbols length i.e. satisfy regexp `^[-a-zA-Z0-9_]{2,}$`).

If you want to use namespace options for channel - you must include namespace name into
channel name with `:` as separator:

`public:messages`

`gossips:messages`

Where `public` and `gossips` are namespace names from project `namespaces`.

All things together here is an example of `config.json` which includes registered project with all options set and 2 additional namespaces in it:

```javascript
{
    "secret": "very-long-secret-key",
    "api_key": "secret-api-key",
    "anonymous": true,
    "publish": true,
    "presence": true,
    "join_leave": true,
    "history_size": 10,
    "history_lifetime": 30,
    "namespaces": [
        {
          "name": "public",
          "publish": true,
          "anonymous": true,
          "history_size": 10,
          "history_lifetime": 300,
          "history_recover": true
        },
        {
          "name": "gossips",
          "presence": true,
          "join_leave": true
        }
    ]
}
```

Channel `news` will use globally defined channel options.

Channel `public:news` will use `public` namespace's options.

Channel `gossips:news` will use `gossips` namespace's options.

There is no inheritance in channel options and namespaces – so if for example you defined `presence: true` on top level of configuration and then defined namespace – that namespace won't have presence enabled - you must enable it for namespace explicitly. 

### Advanced configuration

Centrifugo has some options for which default values make sense for most applications. In many case you don't need (and you really should not) change them. This chapter is about such options.

#### client_channel_limit

Default: 128

Sets maximum number of different channel subscriptions single client can have.

#### channel_max_length

Default: 255

Sets maximum length of channel name.

#### channel_user_connection_limit

Default: 0

Maximum number of connections from user (with known user ID) to Centrifugo node. By default - unlimited.

#### client_request_max_size

Default: 65536

Maximum allowed size of request from client in bytes.

#### client_queue_max_size

Default: 10485760

Maximum client message queue size in bytes to close slow reader connections. By default - 10mb.

#### client_anonymous

Default: false

Enable mode when all clients can connect to Centrifugo without JWT connection token. In this case all connections without token will be treated as anonymous (i.e. with empty user ID) and only can subscribe to channels with `anonymous` option enabled.

#### sockjs_heartbeat_delay

Default: 25

Interval in seconds how often to send SockJS h-frames to client.

#### websocket_compression

Default: false

Enable websocket compression, see chapter about websocket transport for more details.

#### gomaxprocs

Default: 0

By default Centrifugo runs on all available CPU cores. If you want to limit amount of cores Centrifugo can utilize in one moment use this option.

### Advanced endpoint configuration.

After you started Centrifugo you have several endpoints available. As soon as you have not provided any extra options you have 3 endpoints by default.

#### Default endpoints.

The main endpoint is raw Websocket endpoint to serve client connections that use pure Websocket protocol:

```
ws://localhost:8000/connection/websocket
```

Then there is SockJS endpoint - it's needed to serve client connections that use SockJS library:

```
http://localhost:8000/connection/sockjs
```

And finally you have API endpoint to `publish` messages to channels (and execute other available API commands):

```
http://localhost:8000/api
```

By default all endpoints work on port `8000`. You can change it using `port` option:

```
{
    "port": 9000
}
```

In production setup you will have your domain name in endpoint addresses above instead of `localhost`. Also if your Centrifugo will be behind proxy or load balancer software you most probably won't have ports in your endpoint addresses. What will always be the same as shown above are URL paths: `/connection/sockjs`, `/connection/websocket`, `/api`.

Let's look at possibilities to tweak available endpoints.

#### Admin endpoints.

First is enabling admin endpoints:

```
{
    ...
    "admin": true,
    "admin_password": "password",
    "admin_secret": "secret"
}
```

This makes the following endpoint available: http://localhost:8000

At this address you will see admin web interface. You can log into it using `admin_password` value shown above.

#### Debug endpoints.

Next, when Centrifugo started in debug mode some extra debug endpoints become available. To start in debug mode add `debug` option to config:

```
{
    ...
    "debug": true
}
```

And endpoint:

```
http://localhost:8000/debug/pprof/
```

– will show you useful info about internal state of Centrifugo instance. This info is especially helpful when troubleshooting. See [wiki page](https://github.com/centrifugal/centrifugo/wiki/Investigating-performance-issues) for more info.

#### Healthcheck endpoint

New in v2.1.0

Use `health` boolean option (by default `false`) to enable healthcheck endpoint which will be available on path `/health`. Also available over command-line flag:

```bash
./centrifugo -c config.json --health
```

#### Custom internal ports

We strongly recommend to not expose API, admin, debug and prometheus endpoints to Internet. The following Centrifugo endpoints are considered internal:

* API endpoint (`/api`) - for HTTP API requests
* Admin web interface endpoints (`/`, `/admin/auth`, `/admin/api`) - used by web interface
* Prometheus endpoint (`/metrics`) - used for exposing server metrics in Prometheus format 
* Healthcheck endpoint (`/health`) - used to do healthchecks
* Debug endpoints (`/debug/pprof`) - used to inspect internal server state

It's a good practice to protect those endpoints with firewall. For example you can do this in `location` section of Nginx configuration.

Though sometimes you don't have access to per-location configuration in your proxy/load balancer software. For example when using Amazon ELB. In this case you can change ports on which your internal endpoints work.

To run internal endpoints on custom port use `internal_port` option:

```
{
    ...
    "internal_port": 9000
}
```

So admin web interface will work on address:
 
```
http://localhost:9000
```

Also debug page will be available on new custom port too:

```
http://localhost:9000/debug/pprof/
```

The same for API and prometheus endpoint.
