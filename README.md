# An alternative Mongodb driver for Elixir
[![Build Status](https://travis-ci.org/zookzook/elixir-mongodb-driver.svg?branch=master)](https://travis-ci.org/zookzook/elixir-mongodb-driver)
[![Hex.pm](https://img.shields.io/hexpm/v/mongodb_driver.svg)](https://hex.pm/packages/mongodb_driver)
[![Hex.pm](https://img.shields.io/hexpm/dt/mongodb_driver.svg)](https://hex.pm/packages/mongodb_driver)
[![Hex.pm](https://img.shields.io/hexpm/dw/mongodb_driver.svg)](https://hex.pm/packages/mongodb_driver)
[![Hex.pm](https://img.shields.io/hexpm/dd/mongodb_driver.svg)](https://hex.pm/packages/mongodb_driver)

This is an alternative development from the [original](https://github.com/ankhers/mongodb), which was the starting point
and already contained very nice code.

The [Documentation](https://hexdocs.pm/mongodb_driver/readme.html) is online, but currently not up to date. 
This will be done as soon as possible. In the meantime, look in the source code. Especially 
for the individual options.  

## Motivation

  * [x] I have made a number of changes to understand how the driver works. For example, I reduced cursor modules to just one cursor and
        replaced some op code calls with command calls.
  * [x] Simplify code: remove raw_find (raw_find called from cursors, raw_find called with "$cmd"), so raw_find is more calling a command than a find query.
  * [x] Better support for new MongoDB version, for example the ability to use views
  * [x] Upgraded to ([DBConnection 2.x](https://github.com/elixir-ecto/db_connection))
  * [x] Removed depreacated op codes ([See](https://docs.mongodb.com/manual/reference/mongodb-wire-protocol/#request-opcodes))
  * [x] Added `op_msg` support ([See](https://docs.mongodb.com/manual/reference/mongodb-wire-protocol/#op-msg))
  * [ ] Add support for driver sessions ([See](https://github.com/mongodb/specifications/blob/master/source/sessions/driver-sessions.rst))
  * [ ] Add support driver transactions ([See](https://github.com/mongodb/specifications/blob/master/source/transactions/transactions.rst))
  * [ ] Add support for `op_compressed` ([See](https://github.com/mongodb/specifications/blob/master/source/compression/OP_COMPRESSED.rst))
  * [ ] Because the driver is used in production environments, quick adjustments are necessary.

## Features

  * Supports MongoDB versions 3.2, 3.4, 3.6, 4.0
  * Connection pooling ([through DBConnection 2.x](https://github.com/elixir-ecto/db_connection))
  * Streaming cursors
  * Performant ObjectID generation
  * Follows driver specification set by 10gen
  * Safe (by default) and unsafe writes
  * Aggregation pipeline
  * Replica sets
  * Support for SCRAM-SHA-256 (MongoDB 4.x)
  * Support for change streams api ([See](https://github.com/mongodb/specifications/blob/master/source/change-streams/change-streams.rst))


## Data representation

    BSON                Elixir
    ----------          ------
    double              0.0
    string              "Elixir"
    document            [{"key", "value"}] | %{"key" => "value"} (1)
    binary              %BSON.Binary{binary: <<42, 43>>, subtype: :generic}
    object id           %BSON.ObjectId{value: <<...>>}
    boolean             true | false
    UTC datetime        %DateTime{}
    null                nil
    regex               %BSON.Regex{pattern: "..."}
    JavaScript          %BSON.JavaScript{code: "..."}
    integer             42
    symbol              "foo" (2)
    min key             :BSON_min
    max key             :BSON_max

1) Since BSON documents are ordered Elixir maps cannot be used to fully represent them. This driver chose to accept both maps and lists of key-value pairs when encoding but will only decode documents to lists. This has the side-effect that it's impossible to discern empty arrays from empty documents. Additionally the driver will accept both atoms and strings for document keys but will only decode to strings.

2) BSON symbols can only be decoded.

## Usage

### Installation:

Add `mongodb_driver` to your mix.exs `deps` and `:applications`.

```elixir
def application do
  [applications: [:mongodb_driver]]
end

defp deps do
  [{:mongodb_driver, "~> 0.5.3"}]
end
```

Then run `mix deps.get` to fetch dependencies.

```elixir
# Starts an unpooled connection
{:ok, conn} = Mongo.start_link(url: "mongodb://localhost:27017/db-name")

# Gets an enumerable cursor for the results
cursor = Mongo.find(conn, "test-collection", %{})

cursor
|> Enum.to_list()
|> IO.inspect
```

### Connection pooling
The driver supports pooling by DBConnection (2.x). By default `mongodb_driver` will start a single 
connection, but it also supports pooling with the `:pool_size` option. For 3 connections add the `pool_size: 3` option to `Mongo.start_link` and to all 
function calls in `Mongo` using the pool:

```elixir
# Starts an pooled connection
{:ok, conn} = Mongo.start_link(url: "mongodb://localhost:27017/db-name", pool_size: 3)

# Gets an enumerable cursor for the results
cursor = Mongo.find(conn, "test-collection", %{})

cursor
|> Enum.to_list()
|> IO.inspect
```

If you're using pooling it is recommend to add it to your application supervisor:

```elixir
def start(_type, _args) do
  import Supervisor.Spec

  children = [
    worker(Mongo, [[name: :mongo, database: "test", pool_size: 3]])
  ]

  opts = [strategy: :one_for_one, name: MyApp.Supervisor]
  Supervisor.start_link(children, opts)
end
```

Due to the mongodb specification, an additional connection is always set up for the monitor process.

### Replica Sets

To connect to a Mongo cluster that is using replica sets, it is recommended to use the `:seeds` list instead 
of a `:hostname` and `:port` pair.

```elixir
{:ok, pid} = Mongo.start_link(database: "test", seeds: ["hostname1.net:27017", "hostname2.net:27017"])
```

This will allow for scenarios where the first `"hostname1.net:27017"` is unreachable for any reason 
and will automatically try to connect to each of the following entries in the list to connect to the cluster.

### Auth mechanisms

For versions of Mongo 3.0 and greater, the auth mechanism defaults to SCRAM. 
If you'd like to use [MONGODB-X509](https://docs.mongodb.com/manual/tutorial/configure-x509-client-authentication/#authenticate-with-a-x-509-certificate) 
authentication, you can specify that as a `start_link` option.

```elixir
{:ok, pid} = Mongo.start_link(database: "test", auth_mechanism: :x509)
```

### AWS, TLS and Erlang SSL ciphers

Some MongoDB cloud providers (notably AWS) require a particular TLS cipher that isn't enabled 
by default in the Erlang SSL module. In order to connect to these services,
you'll want to add this cipher to your `ssl_opts`: 

```elixir
{:ok, pid} = Mongo.start_link(database: "test", 
      ssl_opts: [
        ciphers: ['AES256-GCM-SHA384'],
        cacertfile: "...",
        certfile: "...")
      ]
)
```
### Change streams

Change streams exist in replica set and cluster systems and tell you about changes to collections. 
They work like endless cursors.
The special thing about the change streams is that they are resumable. In the case of a resumable error, 
no exception is made, but the cursor is re-scheduled at the last successful location. 
The following example will never stop, 
so it is a good idea to use a process for change streams. 

```
seeds = ["hostname1.net:27017", "hostname2.net:27017", "hostname3.net:27017"]
{:ok, top} = Mongo.start_link(database: "my-db", seeds: seeds, appname: "getting rich")
cursor =  Mongo.watch_collection(top, "accounts", [], fn doc -> IO.puts "New Token #{inspect doc}" end, max_time: 2_000 )  
cursor |> Enum.each(fn doc -> IO.puts inspect doc end)
```

An example with a spawned process that sends message to the monitor process: 

```  
def for_ever(top, monitor) do
    cursor = Mongo.watch_collection(top, "users", [], fn doc -> send(monitor, {:token, doc}) end)
    cursor |> Enum.each(fn doc -> send(monitor, {:change, doc}) end)
end

spawn(fn -> for_ever(top, self()) end)
```
### Examples

Using `$and`

```elixir
Mongo.find(:mongo, "users", %{"$and" => [%{email: "my@email.com"}, %{first_name: "first_name"}]})
```

Using `$or`

```elixir
Mongo.find(:mongo, "users", %{"$or" => [%{email: "my@email.com"}, %{first_name: "first_name"}]})
```

Using `$in`

```elixir
Mongo.find(:mongo, "users", %{email: %{"$in" => ["my@email.com", "other@email.com"]}})
```

## Contributing

The SSL test suite is enabled by default. You have two options. Either exclude
the SSL tests or enable SSL on your Mongo server.

### Disable the SSL tests

`mix test --exclude ssl`

### Enable SSL on your Mongo server

```bash
$ openssl req -newkey rsa:2048 -new -x509 -days 365 -nodes -out mongodb-cert.crt -keyout mongodb-cert.key
$ cat mongodb-cert.key mongodb-cert.crt > mongodb.pem
$ mongod --sslMode allowSSL --sslPEMKeyFile /path/to/mongodb.pem
```

* For `--sslMode` you can use one of `allowSSL` or `preferSSL`
* You can enable any other options you want when starting `mongod`

## License

Copyright 2015 Eric Meadows-Jönsson and Justin Wood

Copyright 2019 Michael Maier


Licensed under the Apache License, Version 2.0 (the "License");
you may not use this file except in compliance with the License.
You may obtain a copy of the License at

    http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.
