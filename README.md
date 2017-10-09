# OTP Message Broker

## Overview

Universal OTP message broker features:
* create channels (pub/sub)
* pools (workers queue)
* mixing it (pool of channels, channel of pools... etc.)
* send messages with specify priority of delivering messages (range: 1..10)
* pool has 3 balance methods: rr(round robin), hash (by erlang:phash(Message, lenth(Pool))), random
* defer message delivering in case of
    - exceed the queue limit (10000) and receiver has the 'true' in 'defer' option.
    - client has 'offline' state and the 'defer' option is set to 'true'
* 'async' Client option allows you control the delivery process. Default value of this option is 'true'.

## Distributed mode

Set via environment value of **'master'** to run it in slave mode.
```erlang
% on mxnode01@127.0.0.1
application:start(mx).
```
```erlang
% on mxnode02@127.0.0.1
application:load(mx),
application:set_env(mx, master, 'mxnode01@127.0.0.1'),
application:start(mx).
```

Call **mx:nodes()** to get the list of mx cluster nodes.

```erlang
> (mxnode01@127.0.0.1)2> mx:nodes().
['mxnode01@127.0.0.1','mxnode02@127.0.0.1']
```


## Examples

```erlang
% Client has higest priority by default (priority = 1)
{clientkey, Client1Key} = mx:register(client, "Client1"),
{clientkey, Client2Key} = mx:register(client, "Client2", [{priority, 8}]),
{clientkey, Client3Key} = mx:register(client, "Client3", [{async, false}, {defer, true}]),
{clientkey, Client4Key} = mx:register(client, "Client4"),

% register channel with default priority (5)
{channelkey, Channel1Key} = mx:register(channel, "Channel1", Client4Key),
ok = mx:subscribe(Client1Key, Channel1Key),
% just for example try to subscribe one more time
{already_subscribed, Client1Key} = mx:subscribe(Client1Key, Channel1Key),

ok = mx:subscribe(Client2Key, Channel1Key),
ok = mx:subscribe(Client3Key, Channel1Key),

mx:send(Channel1Key, "Hello, Channel1!").

% register pool with default balance method is 'rr' - round robin
%             default priority (5)
{poolkey, Pool1Key} = mx:register(pool, "Pool1", Client4Key),
mx:join(Client1Key, Pool1Key),
mx:join(Client2Key, Pool1Key),
mx:join(Client3Key, Pool1Key),

% create lowest priority channel and pool by Client2
{channelkey, LPCh1} = mx:register(channel, "LP Channel1", Client2Key, [{priority, 10}]),
{poolkey, LPPl1}    = mx:register(pool, "LP Pool1", Client2Key, [{priority, 10}]),

% create highest priority channel and pool by Client3
{channelkey, HPCh1} = mx:register(channel, "HP Channel1", Client2Key, [{priority, 1}]),
{poolkey, HPPl1}    = mx:register(pool, "HP Pool1", Client2Key, [{priority, 1}]),

% high priority pool with 'hash' balance
{poolkey, HP_Hash_Pl}    = mx:register(pool, "HP Pool (hash)", Client2Key, [{priority, 1}, {balance, hash}]),

% pool with random balance
{poolkey, Rand_Pl}    = mx:register(pool, "Pool (random) ", Client2Key, [balance, hash}]),

```

## API

### local use

* Create client/channel/pool

    ```erlang 
    mx:register(client, Name)
    mx:register(client, Name, Opts)
    ```
       _Name_ - list or binary
       _Opts_ - proplists
    
    returns: `[{clientkey, Key}| {duplicate, Key}]`
    
       _Key_ - binary

    **mx:register(channel, Name, ClientKey)**
    **mx:register(channel, Name, ClientKey, Opts)**
          Name - list or binary
          Opts - proplists
          ClientKey - binary
          returns: {channelkey, Key}
                   {duplicate, Key}

    **mx:register(pool, Name, ClientKey)**
    **mx:register(pool, Name, ClientKey, Opts)**
          Name - list or binary
          Opts - proplists
          ClientKey - binary
          returns: {poolkey, Key}
                   {duplicate, Key}

* Delete client/channel/pool
    **mx:unregister(Key)**

* Set online/offline state
    **mx:online(ClientKey, Pid)**
    **mx:offline(ClientKey)**

* Work with channel/pool
    **mx:subscribe(Key, Channel)**
    **mx:unsubscribe(Key, Channel)**
          Key - binary (ClientKey, ChannelKey, PoolKey)
          Channel - channel name or channel key

    **mx:join(Key, Pool)**
    **mx:leave(key, Pool)**
          Key - binary (ClientKey, ChannelKey, PoolKey)
          Pool - pool name or pool key

* Set options for client/channel/pool
    **mx:set(Key, Opts)**
          Key - binary (ClientKey, ChannelKey, PoolKey)
          Opts - proplists

* Sending message
  **mx:send(ClientKey, Message)**
  **mx:send(ChannelKey, Message)**
  **mx:send(PoolKey, Message)**

* Owning Pool/Channel
  **mx:own(Key, ClientKey)**
          Key - binary (ChannelKey, PoolKey)

  **mx:abandon(Key, ClientKey)**
          Key - binary (ChannelKey, PoolKey)
          orphan Pool/Channel will unregister automaticaly

* Clear deferred messages
    **mx:flush(Key)**
        Key - binary (ClientKey, ChannelKey, PoolKey)
        all - truncate the 'deferred' table

* Info
    **mx:nodes()**
        show MX cluster nodes

    **mx:info(Key)**
          Key - binary (ClientKey, ChannelKey, PoolKey)

    **mx:info(Key, Name)**
          Key - binary (ClientKey, ChannelKey, PoolKey)
          Name - field name

    **mx:relation(Key)**
          Key - binary (ChannelKey, PoolKey)
          shows list of Clients are subscribed/joined to.


### remote use

use **gen_server:call(MX, Message)**, where the **Message** is one of the listed values below:
- {register_client, Client}
- {register_client, Client, Opts}
- {register_channel, ChannelName, ClientKey}
- {register_channel, ChannelName, ClientKey, Opts}
- {register_pool, PoolName, ClientKey}
- {register_pool, PoolName, ClientKey, Opts}
- {unregister, Key}
- {online, ClientKey, Pid}
- {offline, ClientKey}
- {subscribe, Client, To}
- {unsubscribe, Client, From}
- {join, Client, To}
- {leave, Client, From}
- {send, To, Message}
- {own, Key, ClientKey}
- {abandon, Key, ClientKey}
- {info, Key}
- {info, {Key, Name}}
- {relation, Key}
- {set, Key, Opts}
- nodes

```erlang
> (mxnode02@127.0.0.1)2> gen_server:call({mx, 'mxnode01@127.0.0.1'}, nodes).
['mxnode02@127.0.0.1','mxnode01@127.0.0.1']
```

## Testing

There are only common tests (CT) are implemented with some limited set of cases

### Channel tests (pub/sub).

Sequentualy running:

  * [x] 1 client (subscriber) recieves 1 messages
  * [x] 1 client - 1000 messages
  * [x] 1000 clients - 1 messages
  * [x] 1000 clients - 10 messages

Parallel: 
  * [ ] 1000 clients - 10 messages (10 processes)

Priority delivering:
  * 1 client recieves 1000 messages with 10 different priorities: 
    * [ ] 100 messages with priority 1
    * [ ] 100 messages with priority 2
    * [ ] ...
    * [ ] 100 messages with priority 10
    

### Pool tests (worker queue)

Sequentualy running:

  * [ ] 1 client (sender) sends 1 messages to 1 worker
  * [ ] 1 client - 1000 messages - 2 workers (round robin)
  * [ ] 1 client - 1000 messages - 2 workers (hash)
  * [ ] 1 client - 1000 messages - 2 workers (random)
  * [ ] 1000 clients - 1 messages - 4 workers
  * [ ] 1000 clients - 1000 messages - 4 workers

Parallel: _(not implemented)_
  * [ ] 1000 clients - 10 messages (10 processes)

### Run the testing
1. Run MX application as standalone application

```shell
$ make run
```

2. Run "Common Tests"

```shell
$ make ct
```


