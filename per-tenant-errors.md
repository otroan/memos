# Per-tenant error counters in VPP

[TLDR] _Current thinking is that error counters are per-node, not per-tenant per-node. Only a few nodes are tenant aware, and those should be extended with specific per-tenant counters. While splitting the regular per-node error counters to per-tenant could have some value in identifying tenants with problems, it does not seem that the added number of counters and changes to APIs are worth it._

In VPP we have error counters, simple counters and combined counters. These are all two dimensional. Simple and combined counters are dimensioned
by thread index and a user selected index, typically software interface index. The error counters use the dimensions of thread index and error index.
With individual error counter symlinked into the 2d array.
Error counters are a simple u64 packet counter.

There is a need to split these counters across a 3rd dimension. Some sort of additional context, typically tenant.
The context has to be set in in the vlib_buffer_t structure.

## Implementation choices
Today we have the global /node/errors. Which is a two dimensional vector of thread index by error code index.
Alternative 1. Create /node/errors/<n>. Which are replicas of the global error 2d-vector. One per tenant.
Alternative 2. Make /node/errors 3-dimensional. Instead of a vector of scalars make it a vector of simplevec. {thread-index, context, error code}.
Requires changes to all stat clients. And application uses.

Alternative 2 will require changes to all stats clients, while alternative 1 uses existing data structures.

With all plugins loaded there are about 3000 error counters. Assuming 500 counters x 8 thread x 1000 active tenants = 4M counters. ~32MB.
It may not be worth supporting symlinks per counter, so a better approach may be to provide a names array that maps error counter name to index.

How should the feature be configurable?
 - #ifdef and feature enablement through cmake
 - User-configurable and replace error-node with per-tenant node
 - Just an if(feature_flag) in error-node

Where only the first option allows us to selectively use the extra two bytes in vlib_buffer_t.

In the punt/drop node the error counters are incremented directly from a graph node.
There is also an API vlib_error_count(vlib_main_t * vm, uword node_index, uword counter, uword increment), which does not have 
access to vlib_buffer_t. How should that get access to vnet_buffer(b)->tenant_id?
If signature is changed, it's called > 100 times in the code base.

## Assumptions
- Space has to be found vlib_buffer_t. A u16 is likely enough.
- Both the global error counters and the per-tenant counter will be incremented. Ensuring the total counter is unchanged with the introduction of per-tenant counters.
- Which "tenant" a packet belonged to has to be determined by a node in the graph, nodes running prior to this node will use tenant index 0.
- Processing nodes using a new buffer will not by default, they would have to be extended, carry the 3rd dimension index with them.
  Any such packets would be counted against index 0.

## Actions
1. Find a u16 in vlib_buffer_t which is guaranteed to be untrampled through graph traversal
2. Add /errors/names array
3. Per tenant error support in error-node.
4. API to generate the error array for a new tenant. Needs to be called by tenant-aware initialisation code.

## Simple and combined counters

Per-tenant counters for the simple and combined counters are already supported.
A 2d vector indexed by thread and tenant id. There is a bit of boilerplate required to also have per-tenant symlinks, like the below.
Here is a tool that auto-generates the boilerplate based on a counter definition.
[[https://github.com/nataasvpp/nataasvpp/blob/main/tools/counters.py]]

```
/vcdp/tenant/created-sessions
/vcdp/tenant/removed-sessions
/vcdp/tenant/reused-sessions
/vcdp/tenant/rx-octets-and-pkts
/vcdp/tenant/tx-octets-and-pkts
/vcdp/nat/port-alloc-retries-pkts
/vcdp/nat/port-alloc-failures-pkts
/vcdp/nat/rx-octets-and-pkts
/vcdp/nat/tx-octets-and-pkts

/vcdp/tenants/0/created-sessions
/vcdp/tenants/0/removed-sessions
/vcdp/tenants/0/reused-sessions
/vcdp/tenants/0/rx-octets-and-pkts
/vcdp/tenants/0/tx-octets-and-pkts
/vcdp/tenants/1000/created-sessions
/vcdp/tenants/1000/removed-sessions
/vcdp/tenants/1000/reused-sessions
/vcdp/tenants/1000/rx-octets-and-pkts
/vcdp/tenants/1000/tx-octets-and-pkts
/vcdp/tenants/2000/created-sessions
/vcdp/tenants/2000/removed-sessions
/vcdp/tenants/2000/reused-sessions
/vcdp/tenants/2000/rx-octets-and-pkts
/vcdp/tenants/2000/tx-octets-and-pkts
/vcdp/nats/foobar/port-alloc-retries-pkts
/vcdp/nats/foobar/port-alloc-failures-pkts
/vcdp/nats/foobar/rx-octets-and-pkts
/vcdp/nats/foobar/tx-octets-and-pkts
```

Counter definition in JSON

```
[
	{
		"description": "Per-tenant packet statistics",
		"type": "combined",
		"unit": "octets-and-pkts",
		"symlink": true,
		"prefix": "/vcdp/tenant",
		"symlink_prefix": "/vcdp/tenants",
		"counter": [
			"rx",
			"tx"
		]
	},
	{
		"description": "Per-tenant session statistics",
		"type": "simple",
		"unit": "sessions",
		"symlink": true,
		"prefix": "/vcdp/tenant",
		"symlink_prefix": "/vcdp/tenants",
		"counter": [
			"created",
			"removed",
			"reused"
		]
	}
]
```

## Investigate: Current behaviour

error drop:
```
      error[0] = b[0]->error;

      c_index = counter_index (vm, error[0]);
      if (c_index != CLIB_U32_MAX)
      	em->counters[error[0]] += count;
```

Each node has its own enum of error code. Those are mapped into global error codes.
The function of counter_index() is to map the node's local error name space into the global one.
It's unclear of why this is needed. Since each graph nodes sets the global error index already.

As: b->error = node->errors[NODE_SPECIFIC_ERROR_CODE];
And not: b->error = NODE_SPECIFIC_ERROR_CODE;


```
always_inline u32
counter_index (vlib_main_t * vm, vlib_error_t e)
{
  vlib_node_t *n;
  u32 ci, ni;

  ni = vlib_error_get_node (&vm->node_main, e);
  n = vlib_get_node (vm, ni);

  ci = vlib_error_get_code (&vm->node_main, e);
  if (ci >= n->n_errors)
    return CLIB_U32_MAX;

  ci += n->error_heap_index;

  return ci;
}
```

