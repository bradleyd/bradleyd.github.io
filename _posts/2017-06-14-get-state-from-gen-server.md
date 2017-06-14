---
layout: post
title: "Get state from a running GenServer"
date: 2017-06-14
---

![sad_connection](/images/sad_cloud1600.png){:height="175px" width="150px"}

There are times when an application is not acting how I would expect.  For example, we had an Elixir application with a DB connection that appeared to be up but not processing requests.

Fortunately, the Erlang and Elixir ecosystem has got you covered.  In this code below, we create a `postgres` connection and store the connection's pid in the GenServer state.

```elixir
# code emitted for brevity
  defmodule State do
    defstruct pid: :nil, max_connections: 20
  end

  def start_link do
    state = []
    GenServer.start_link(__MODULE__, state, name: __MODULE__)
  end

  def init(_state) do
    db = SensorHistoryQueueWorker.Config.lookup(:postgres)
    {:ok, pid} = Postgrex.start_link(hostname: db.host, username: db.username, password: db.password, database: db.database,
     pool: DBConnection.Poolboy, pool_size: db.max_connections, extensions: [{Postgrex.Extensions.JSON, library: Poison}])
    Process.monitor(pid)
    {:ok, %State{pid: pid, max_connections: db.max_connections}}
  end
# code emitted for brevity
```

The important thing here is the pid of the DB connection.  From a remote console, we first get the pid of the GenServer.  Once we have the pid, we can use  [get_state/1](http://erlang.org/doc/man/sys.html#get_state-1) to fetch state.

```
iex(sensor_history_queue_worker@127.0.0.1)3> pid = Process.whereis(SensorHistoryQueueWorker.Producer.Postgres)
iex(sensor_history_queue_worker@127.0.0.1)4> state = :sys.get_state(pid)
%SensorHistoryQueueWorker.Producer.Postgres.State{max_connections: 20, pid: #PID<0.17817.15>}
```

With the knowledge of the DB connection pid, we can check if it is alive? or actually connected

```
iex(sensor_history_queue_worker@127.0.0.1)5> Process.alive?(state.pid)
true
iex(sensor_history_queue_worker@127.0.0.1)6> Postgrex.query(state.pid, "select 1", [], [pool: DBConnection.Poolboy, pool_size: 30])
```

Next time I will show how to use `dbg` and `Process.info/1` to learn more about the process.

