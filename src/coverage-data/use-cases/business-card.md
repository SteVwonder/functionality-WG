
### Brief Description

Multi-process communication libraries, such as MPI, need to establish communication channels between a set of those processes. Each process needs to share connectivity information (a.k.a. Business Cards) with the set of processes with which they will communicate before communication channels can be established. The runtime environment must provide a mechanism for the efficient exchange of this connectivity information. Additional information about the current state of the job (e.g., number of processes globally and locally) and of how the process was started (e.g., process binding) are also helpful during this activity.

### Use Case Details

**Note**: The Instant-On wire-up mechanism is a separate, related use case.

Multi-process communication libraries, such as MPI, need to establish communication channels between a set of those processes. Each process needs to share connectivity information (a.k.a. Business Cards) with all other processes before communication channels can be established. This connectivity information may take the form of one or more unique strings that allow a different process to establish a communication channel with the originator.

Each process provides their business card to PMIx via one or more `PMIx_Put` operations to store the tuple of `{UID, key, value}`. The `UID` is the unique name for this process in the PMIx universe (i.e., `namespace` and `rank`). The `key` is a unique key associated with this process that other processes can reference (note that since the `UID` is also associated with the `key` there is no need to make the `key` uniquely named per process). The `value` is the string representation of the connectivity information.

Some business card information is meant for remote processes (e.g., TCP or InfiniBand addresses) while others are meant only for local processes (e.g., shared memory information). As such a `scope` should be associated with the `PMIx_Put` operation to differentiate this intention.

The `PMIx_Put` operations may be cached local to the process. Once all `PMIx_Put` operations have been called each process should call `PMIx_Commit` to push those values to the local PMIx server. Note that in a multi-library configuration each library may `PMIx_Put` then `PMIx_Commit` values. As a result there may be multiple `PMIx_Commit` calls before a Business Card Exchange is activated.

After the completion of the `PMIx_Commit` call then the data `PMIx_Put` by this process is immediately available to be retrieved by other processes. There are two mechanisms by which a remote process can retrieve this information. Namely a "**Full Business Card Exchange**" (a.k.a. "full modex") and a "**Direct Business Card Exchange**" (a.k.a. "direct modex"). The consumer of the business card information may prefer one method over the other depending on how they choose to wireup.

The `PMIx_Fence` operation is collective over the set of processes specified in the argument set. That allows for the collective to span a subset of a namespace or multiple namespaces. The `PMIx_Fence` behaves as a barrier operation without further parameterization. It can be helpful in coordinating when business card information is available between a set of processes.

In a "**Full Business Card Exchange**" (a.k.a. "full modex") all business card information is exchanged between a specified set of processes making it immediately locally available to each process. A `PMIx_Fence` is called after the `PMIx_Commit` completes to collectively exchange this information between the specified processes. The `PMIx_Fence` operation takes an additional parameter indicating that a full exchange of business card information is requested of the runtime environment. A process can then use the `PMIx_Get` call to retrieve the key/value pairs from the remote process from the local copy of the data. The "local" nature of the storage is defined by the PMIx library.

In a "**Direct Business Card Exchange**" (a.k.a. "direct modex") business card information is pulled to a process when it makes the first request for such information by making a `PMIx_Get` call. A `PMIx_Fence` is not necessary in this type of exchange, but may be useful in synchronizing the availablibility of data between the processes. The `PMIx_Get` call will first look locally for the requested `{UID, key}` pair. If it is not found locally then a "direct modex" request (see `pmix_server_dmodex_req_fn_t` and `PMIx_server_dmodex_request` on the server side) is sent to the resource manager on the node hosting that process to retrieve the current set of key/value pairs for that process. The `PMIx_Get` completes once that information is returned to the calling process. The caller can specify an attribute (`PMIX_IMMEDIATE`) to `PMIx_Get` to force the call to only look for the data locally and not trigger the "direct modex" request.

Note that it is possible that the remote process has not `PMIx_Put` and `PMIx_Commit`ed the key/value pair at the moment another process tries to access that information. The resource manager may return the currently available key/value pairs immediately or wait for some period of time. The caller can specify an attribute (`PMIX_TIMEOUT`) to `PMIx_Get` to specify the amount of time the server should wait for this information. 

The "Full Business Card Exchange" is helpful for tightly communicating applications since they exchange all of the reuqiqred information at once instead of on-demand. The "Direct Business Card Exchange" is helpful for sparsely communicating applications since only the information required by the process is accessed. As a result, the direct method may also have memory consuption advantages particularly at scale.

Additional information about the current state of the job (e.g., number of processes globally and locally) and of how the process was started (e.g., process binding) are also helpful. This "job level" information must be available immediately after `PMIx_Init` without the need for any explicit synchronization.

The number of processes globally in the namespace (`PMIX_JOB_SIZE`) and this process's rank within that namespace (`PMIX_RANK`)is important to know before establishing the Business Card information to best allocate resources.

The number of processes local to the node and this process's local rank is important to know before establishing the Business Card information to help the caller determine the scope of the put operation. For example, to designate a leader to set up a shared memory segment of the proper size before putting that information into the locally scoped Business Card information.

The number of processes local to a remote node is also helpful to know before establishing the Business Card information. This information is useful to pre-establish local resources before that remote node starts to initiate a connection or to determine the number of connections that need to be advertised in the Business Card when it is sent out.

Note that some of the job level information may change over the course of the job in a dynamic application.

#### Interfaces
```
PMIx_Put
PMIx_Get
PMIx_Commit
PMIx_Fence
PMIx_Init
```

#### Keys

The following job level information is useful to have before establishing Business Card information:
 * `PMIX_NODE_LIST` List of nodes in the job
 * `PMIX_NUM_NODES` Number of nodes in the job
 * `PMIX_NODEID` Node ID where this process is located
 * `PMIX_JOB_SIZE` Number of processes globally in the job
 * `PMIX_PROC_MAP` Mapping of processes to nodes in this job
 * `PMIX_LOCAL_PEERS` List of local processes on this node in this job
 * `PMIX_LOCAL_SIZE` Number of processes local to this node.

For each process this information is also useful (note that any one process may want to access this list of information about any other process in the system):
 * `PMIX_RANK` My global rank in the job
 * `PMIX_LOCAL_RANK` My local rank on this node for this job
 * `PMIX_GLOBAL_RANK` My global rank across all namespaces
 * `PMIX_LOCALITY_STRING` Process binding on this node
 * `PMIX_HOSTNAME` hostname associated with this process (useful for queries about remote processes)

There are other keys that are helpful to have before a synchronization point, this is not meant to be a comprehensive list.

### References

 * PMI v2 [link](https://wiki.mpich.org/mpich/index.php/PMI_v2_API)
 * PMIx: Process management for exascale environments (Nov. 2018) [DOI](https://doi.org/10.1016/j.parco.2018.08.002)
