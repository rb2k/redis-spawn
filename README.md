redis-spawn
===========

An extension to redis-rb to facilitate spawning a redis-server specifically
for your app.

Why?
----

Redis focuses on providing its services to a single application at a time,
and is geared towards providing scalability for whatever application you are
using it for.  It doesn't really expect you to try and use a single server
instance and dataset for more than one application at a time: there is only
very limited access control for the whole server, and no real way of
partitioning the keyspace (you can namespace, but you can't access control a
namespace or any specific subset of keys).  One solution is to run a server
instance for each application and dataset you want to use.  The goal of
redis-spawn is to allow you to manage your own redis-server instances from
within your Ruby application.

Usage
-----

To start a server with default options, simply create a new
Redis::SpawnServer instance:

```ruby
    require 'redis/spawn'

    my_server_instance = Redis::SpawnServer.new
    my_server_instance.pid
    => 7459
```

When a Redis::SpawnServer instance is created, the initialize method sets up
the options for the server, and then starts it (this can be overridden - see
below).  You should then be able to see the server in your process list:

```sh
    $ ps -fp 7459
      PID TTY      STAT   TIME COMMAND
     7459 pts/4    S+     0:00 redis-server /tmp/redis-spawned.7457.config
```

### Default Options

When starting a server with default options, redis-spawn will create a Redis
config file named `/tmp/redis-spawned.[pid of parent process].config`, and
populate it with the default server options. For the most part, these
defaults correspond to the defaults you expect to find in a vanilla
`redis.conf`, with a few notable exceptions:

 * `dir` defaults to `/tmp/redis-spawned.[ppid].data`
 * `logfile` defaults to `"/tmp/redis-spawned.[ppid].log`
 * `unixsocket` defaults to `/tmp/redis-spawned.[ppid].sock`
 * `port` defaults to `0`
 * `bind` defaults to `127.0.0.1`
 * `daemonize` is not set (i.e. server won't daemonize)

In all cases, [ppid] corresponds to the PID of the Ruby process that spawns
the server.

### Connecting

By defaults, spawned servers use unix domain sockets, and don't bind to a
network port. Continuing the above example you can connect as follows:

```ruby
    redis = Redis.new(:path => my_server_instance.socket)
```

Or from the command line:

```sh
    $ redis-cli -s /tmp/redis-spawned.7457.sock
```

### Shutdown and Cleanup

Spawned servers will automatically shutdown when your program exits. You can
also manually shut down:

```ruby
    my_server_instance.shutdown
    => 1
    my_server_instance.pid
    => nil
```

By default, redis-spawn will cleanup the socket, log, and config files that
were automatically created.  The data directory is not removed: you will
need to clean this up yourself if you don't need it (@todo support data dir
removal).

### Server Options

You can set :server_opts parameter when initializing to control server
configuration options. The value of this parameter should be a hash of Redis
config key/value pairs:

```ruby
    my_server_opts = {:port => 6379, :bind => 192.168.0.1}
    my_net_server = Redis::SpawnServer.new(:server_opts => my_server_opts)
```

:server_opts keys are Ruby symbols corresponding to names of Redis config
keys. Underscores in symbol names get translated to dashes in the config
file e.g. :hash_max_zipmap_value corresponds to hash-max-zipmap-value.

Values are expected to be strings or supply strings via #to_s in the usual
way. There is one exception: if the value is an array, the configuration
line will be written multiple times, once for each value of the string. For
example:

```ruby
    :save => ["900 1", "300 10", "60 10000"]
```

becomes

```
    save 900 1
    save 300 10
    save 60 10000
```

### Other options

There are several options which you can pass to Redis::SpawnServer.new,
which are as follows:

#### `:generated_config_file`

This allows you to override the name/path of the autogenerated config file.

#### `:config_file`

Allows you to supply your own pre-existing config file. If you pass this
parameter, redis-spawn will not generate a config file and won't attempt to
clean up any files on shutdown.

#### `:cleanup_files`

This controls which files get automatically cleaned up when the server is
shut down. When setting this parameter, pass an array of symbols
corresponding to the files you want cleaning up:

```ruby
    :cleanup_files => [:socket, :config]
```

The default for this parameter is `[:socket, :log, :config]`, unless the
the `:config_file` parameter is set, in which case the default is to not
clean up any files unless `:cleanup_files` is set explicitly.

At present, there is no built in mechanism for cleaning up the data directory
- you will always need to do this manually.

#### `:start`

This allows you to prevent the server from automatically being started, e.g.

```ruby
    # Don't want to start striaght away
    my_server = Redis::SpawnServer(:start => false)
    # ...
    # Now we're ready to start
    my_server.start
```

Contact and Contributing
------------------------

The homepage for this project is

http://github.com/LichP/redis-spawn

Any feedback, suggestions, etc are very welcome. If you have bugfixes and/or
contributions, feel free to fork, branch, and send a pull request.

Enjoy :-)

Phil Stewart, October 2011
