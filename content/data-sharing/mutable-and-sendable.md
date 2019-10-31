---
title: "Mutable and Sendable"
section: "Data Sharing"
menu:
  toc:
    parent: "data-sharing"
    weight: 40
---

## Problem

Suppose you have a central Supervisor actor that manages a very large data structure that contains the work produced by a collection of Worker actors.  However, the Superviser must also respond to data requests (perhaps from the Workers themselves, or in this case, from another set of Requester actors).  As a result, the Superviser's data structure must be both mutable (to update it in response to feedback from Workers) and sendable (to respond to requests from Requesters).  `iso` is the only refcap that accommodates that, but our Supervisor can't use `iso` here because it must keep a reference to the data structure to continue to respond to incoming updates and requests.  How to proceed?

## Solution

```pony
use mut = "collections"
use "persistent/collections"

type Datum is List[I32]


actor Worker
  let _supervisor: Supervisor

  new create(supervisor: Supervisor) =>
  _supervisor = supervisor
  
  be do_work(id: USize) =>
	let result = [as I32: 1; 2; 3; 4] // initial state array
	// do heavy lifting here; mutate array as needed
	_supervisor.update(id, Lists[I32](result))
	
	
actor Requester
  let _supervisor: Supervisor
  
  new create(supervisor: Supervisor) =>
    _supervisor = Supervisor
    
  be request(id:USize) =>
    _supervisor.get(id, this)

  be receive(id: USize, result: Datum) =>
    // this behavior is called by Supervisor in response to a request
    // use the result here
    
  
actor Supervisor
  let _data: mut.Map[USize, Datum]
  let _default: Datum = Lists[I32]([0; 0; 0; 0])
  
  new create() =>
    _data = mut.Map[USize, Datum](268_435_456) // == 2 ** 28; point being a non-trivially sized data structure, use your imagination
  
  be update(id: USize, new_data: Datum) => 
    _data.update(id, new_data)                 // _data.upsert is perhaps more realistic, but keeping it simple here
    
  be get(id: USize, requester: Requester) =>
    requester.receive(id, _data.get_or_else(id, _default))
```


The Supervisor uses a mutable `ref` Map collection of persistent immutable `val` Lists.  Because the Map is mutable, new data can be easily added or updated without the copying costs required by a persistent Map.  Because we're storing the individual work product (Datum) in persistent immutable Lists, they are refcap `val` which accommodates sharing with Requesters.


## Discussion

Several other solutions might occur to you before the one presented here.  Such as:

Could we use the copying strategy instead?  If the data structure was fully mutable (say a Map of Arrays), we would have to copy the `ref` Array to a `val` Array on every update and every request.  This solution only copies on update, which is a big win, especially if requests materially outnumber updates.

Persistent Maps are sendable and can be updated.  Can we just use that?  Our data structure could be so large that the copying cost of each update to a persistent structure seems prohibitive.

Could we use a `Map iso` data structure?  Well, if we chose an `iso`, the compiler would allow us to both mutate it for updates and send it to a Requester.  However, if we send it to a Requester, the Supervisor no longer has it, and it blocks!  The supervisor can no longer continue to process updates and requests until the `iso` is sent back by the Requester.  The blocking is undesirable, and "sending the iso back" requires additional code.

The pattern presented here combines aspects of copying and persistent data structures.  Each worker creates a copy of the new result during its `do_work` behavior, yet the mutable map is cheaply updated by the Supervisor.  The results are stored in persistent `Lists`, which are `val` and can be shared with other actors.
