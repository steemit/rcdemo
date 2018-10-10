# Resource Credit System developer guide

The goal of this guide is to demystify how resources and RC's work.  The intended audience is developers working on Steem user interfaces,
applications, and client libraries.

### Statelessness

First of all, a word about statelessness.  A great deal of effort has gone into carefully separating stateful from stateless computations.
The reason for this is so that UI's and client libraries can execute stateless algorithms locally.  Local computation is always to be
preferred to RPC-API for performance, stability, scalability and security.

Unfortunately, client library support for RC algorithms is lacking.  This tutorial, and its accompanying script, are intended to
provide guidance to UI and client library maintainers about how to add support for the RC system.

# The RC demo script

To get up and running, I (`theoretical`) have transcribed some key algorithms from C++ to Python.  For example, how many resources
are consumed by a vote transaction?  The `rcdemo` script allows us to find out:

```
>>> from rcdemo import *
>>> count = count_resources(vote_tx, vote_tx_size)
>>> count["resource_state_bytes"]
499232
>>> print(json.dumps(count))
{"resource_count": {"resource_history_bytes": 133, "resource_new_accounts": 0, "resource_market_bytes": 0, "resource_state_bytes": 499232, "resource_execution_time": 0}}
```

The `count_resources()` function is *stateless*.  That means all of the information needed to do the calculation is contained in the transaction itself.  It doesn't
depend on what's happening on the blockchain, or what other users are doing.  [1] [2] [3]

[1] Although it is possible that the calculation will change in future versions of `steemd`, for example to correct the [bug](https://github.com/steemit/steem/issues/2972) where execution time is always reported as zero.

[2] For convenience, some of the constants used in the calculation are exposed by the `size_info` member of `rc_api.get_resource_params()`.  Only a `steemd` version upgrade can change any values returned by `rc_api.get_resource_params()`, so it is probably okay to query that API once, on startup or when first needed, and then cache the result forever.  Or even embed the result of `rc_api.get_resource_params()` in the source code of your library or application.

[3] `rcdemo.py` requires you to also input the transaction size into `count_resources()`.  This is because `rcdemo.py` was created to be a standalone script, without a dependence on any particular client library.  If you are integrating `rcdemo.py` into a client library, you might consider using your library's serializer to calculate the transaction size automatically, so the caller of `count_resources()` doesn't have to specify it.

### Resources

Let's go into details on the different kinds of resources which are limited by the RC system.

- `resource_history_bytes` : Number of bytes consumed by the transaction.
- `resource_new_accounts` : Number of accounts created by the transaction.
- `resource_market_bytes` : Number of bytes consumed by the transaction if it contains market operations.
- `resource_state_bytes` : Number of bytes of chain state needed to support the transaction.
- `resource_execution_time` : An estimate of how long the transaction will take to execute.  Zero for now due to [Steem issue 2972](https://github.com/steemit/steem/issues/2972).

The resources have different scales.  The resources use fixed-point arithmetic where one "tick" of the resource value is a "fractional" value of the resource.  Right now, the resource scales are scattered in different places.  The `count_resources()` result has the following scale:

- `resource_history_bytes` :  One byte is equal to `1`.
- `resource_new_accounts` :   One account is equal to `1`.
- `resource_market_bytes` :   One byte is equal to `1`.
- `resource_state_bytes` :    One byte which must be stored forever is equal to the `steemd` compile-time constant `STATE_BYTES_SCALE`, which is `10000`.  Bytes which must be stored for a bounded amount of time are worth less than `10000`, depending on how long they need to be stored.  The specific constants used in various cases are specified in the `resource_params["size_info"]["resource_state_bytes"]` fields.
- `resource_execution_time` : One nanosecond of CPU time is equal to `1`.  The values are based on benchmark measurements made on a machine similar to `steemit.com` servers.  Some rounding was performed, and a few operations' timings were adjusted to account for additional processing of the virtual operations they cause.

### Resource pool levels

Each resource has a global *pool* which is the number of resources remaining.  The pool code supports fractional resources, the denominator is represented by the `resource_unit` parameter.  So for example, since `resource_params["resource_params"]["resource_market_bytes"]["resource_dynamics_params"]["resource_unit"]` is `10`, a pool level of `15,000,000,000` actually represents `1,500,000,000` bytes.

### Resource credits

The RC cost of each resource depends on the following information:

- How many resources are in the corresponding resource pool
- The global RC regeneration rate, which may be calculated as `total_vesting_shares / (STEEM_RC_REGEN_TIME / STEEM_BLOCK_INTERVAL)`
- The price curve parameters in the corresponding `price_curve_params` object

For convenience, `rcdemo.py` contains an `RCModel` class with all of this information in its fields.

```
>>> print(json.dumps(model.get_transaction_rc_cost( vote_tx, vote_tx_size )))
{"usage": {"resource_count": {"resource_history_bytes": 133, "resource_new_accounts": 0, "resource_market_bytes": 0, "resource_state_bytes": 499232, "resource_execution_time": 0}}, "cost": {"resource_history_bytes": 42136181, "resource_new_accounts": 0, "resource_market_bytes": 0, "resource_state_bytes": 238436287, "resource_execution_time": 0}}
>>> sum(model.get_transaction_rc_cost( vote_tx, vote_tx_size )["cost"].values())
280572468
```

The `model` object created in `rcdemo.py` is an instance of `RCModel` which uses hardcoded values for its pool levels and global RC regeneration rate.  These values were taken from the live network and hardcoded in the `rcdemo.py` source code in late September 2018.  So the RC cost calculation provided out-of-the-box by `rcdemo.py` are approximately correct as of late September 2018, but will become inaccurate as the "live" values drift away from the hardcoded values.  When integrating the `rcdemo.py` code into an application, client library, or another situation where RPC access is feasible, you should understand how your code will query a `steemd` RPC endpoint for current values.  (Some libraries will probably choose to do this RPC automagically, other libraries may want to leave this plumbing to user code.)

### Transaction limits

Suppose an account has 15 Steem Power.  How much can it vote?

```
>>> vote_cost = sum(model.get_transaction_rc_cost( vote_tx, vote_tx_size )["cost"].values())
>>> vote_cost
280572468
>>> vote_cost * total_vesting_fund_steem / total_vesting_shares
138.88697555075086
```

This is the amount of Steem Power (in satoshis) that would be needed by an account to transact once per 5 days (`STEEM_RC_REGEN_TIME`).
Our 15 SP account has 15000 SP, so it would be able to do `15000 / 138`, or about `108`, such transactions per 5 days.

You can regard the number `138` (or `0.138`) as the "cost" of a "standardized" vote transaction.  It plays an analogous role to a
transaction fee in Bitcoin, but it is not exactly a fee.  Because the word "fee" implies giving up a permanent token with a limited,
controlled emission rate.  It is the amount of SP which will allow a user an additional vote transaction every 5 days (but it might
be slightly more or less, if your vote transactions use a slightly different amount of resources.)

### Integrating the demo script

The `rcdemo.py` script is a standalone Python script with no dependencies, no network access, and no transaction serializer.  It is a port
of the algorithms, and a few example transactions for demo purposes.

Eventually, client library maintainers should integrate `rcdemo.py` or equivalent functionality into each Steem client library.  Such integration
depends on the idioms and conventions of each particular client library, for example:

- A client library with a minimalist, "explicit is better than implicit" philosophy may simply rename `rcdemo`,
delete the example code, add a tiny bit of glue code, and give it to clients largely as-is.
- A client library with an automagic, "invisible RPC" philosophy might provide a transaction may make substantial modifications to `rcdemo`
or expose an interface like `get_rc_cost(tx)` which would conveniently return the RC cost of a transaction.  Since the RC cost depends on
chain state, this `get_rc_cost()` function would call `steemd` RPC's (or use cached values) to get additional inputs needed by stateless
algorithms, such as resource pools, `total_vesting_shares`, etc.
- A client library which has a general policy of hard-coding constants from the `steemd` C++ source code might distribute
`rc_api.get_resource_parameters()` as part of its source code, as `rc_api.get_resource_parameters()` results can only change in a new
version of `steemd`.  (Perhaps this kind of thing is somehow automated in the library's build scripts.)
- A client library whose maintainers don't want to be obligated to make new releases when values in `steemd` change as part of a new release,
might instead choose to call `rc_api.get_resource_parameters()`.  Adding extra performance and security overhead in exchange for the
convenience of asking `steemd` to report these values.
- A client library with its own `Transaction` class might choose to implement class methods instead of standalone functions.
- A client library in JavaScript, Ruby or Go might transcribe the algorithms in `rcdemo.py` into a different language.

As you can see, integrating support for the RC system into a Steem client library involves a number of architectural and
technical decisions.
