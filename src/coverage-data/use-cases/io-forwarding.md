### Brief Description

Discusses support for all tools by adding the ability for the RM to forward output from applications to a tool, and to forward stdin from the tool to one or more application processes.

### Use Case Details

Support for tools and debuggers can be extended by adding the ability for the RM to forward output from applications to a tool, and to forward stdin from the tool to one or more application processes. 


1. Forwarding Stdout/Stderr
Once the tool has connected to the target server, it can request that output from a specified set of processes in a given executing application be forwarded to it. In addition, requests to forward output from processes being spawned by the tool can be included in calls to the PMIx_Spawn API. When using PMIx_Spawn API two modes are envisioned:

- PMIX_IOF_COPY // deliver a copy of the output to the tool
- PMIX_IOF_REDIRECT //intercept the output stream and deliver to the requesting tool instead of its current final destination

If PMIx is used to forward stdout/stderr for an already running application PMIx passes a request to the host environment via PMIx_IOF_pull asking that it collect the stdout/stderr from the specified process and forward it to the requestor's specified callback function. The success of this operation depends on the host environment. Typically, the host environment has fork/exec'd the target procs and therefore is already collecting the stdout/stderr from them (e.g., for forwarding to the user).

#### Interfaces
```
PMIx_IOF_pull     // Register to receive output forwarded from a remote process
PMIx_IOF_deregister     // Deregister from output forwarded from a remote process
PMIx_server_IOF_deliver     // Provide a function by which the host RM can pass forwarded IO to the local PMIx server
PMIx_IOF_push     // Push data collected locally (typically from stdin) to target recipients.
```

#### Macros
```
PMIX_FWD_NO_CHANNELS        0x0000
PMIX_FWD_STDIN_CHANNEL      0x0001
PMIX_FWD_STDOUT_CHANNEL     0x0002
PMIX_FWD_STDERR_CHANNEL     0x0004
PMIX_FWD_STDDIAG_CHANNEL    0x0008
PMIX_FWD_ALL_CHANNELS       0x00ff
```

#### Attributes/Directives
```
PMIX_IOF_TAG – output is prefixed with the nspace,rank of the source and a string identifying the channel (stdout, stderr, etc.)
PMIX_IOF_TIMESTAMP – output is marked with the time at which the data was received by the tool (note that this will differ from the time at which it was actually output by the source)
PMIX_IOF_XML_OUTPUT – output is to be formatted in XML
PMIX_IOF_COPY_STREAM -  to get a copy of output streams 
PMIX_IOF_INTERCEPT_STREAM -  to intercept streams  
```

2. Forwarding Stdin
The tool is not necessarily a child of the RM as it may have been started directly from the command line. Thus, provision must be made for the tool to collect its stdin and pass it to the host RM (via the PMIx server) for forwarding. Two methods of support for forwarding of stdin are defined:

- inclusion of PMIX_IOF_PUSH_STDIN in a call to PMIx_Spawn indicating that your stdin is to be forwarded to the specified proc started by the spawn
- external collection directly by the PMIx tool code.  The tool can directly control the targets for the data on a per-call basis through PMIx_IOF_push to begin pushing your stdin to the specified targets

#### Attributes/Directives
```
/*from RFC*/
PMIX_IOF_CACHE_SIZE
"Pmix.iof.csize"
(uint32_t) requested size of the server cache in bytes for each specified channel. By default, the server is allowed (but not required) to drop all bytes received beyond the max size
PMIX_IOF_DROP_OLDEST
"Pmix.iof.old"
(bool) in an overflow situation, drop the oldest bytes to make room in the cache
PMIX_IOF_DROP_NEWEST
"Pmix.iof.new"
(bool) in an overflow situation, drop any new bytes received until room becomes available in the cache (default)
PMIX_IOF_BUFFERING_SIZE
"Pmix.iof.bsize"
(uint32_t) basically controls grouping of IO on the specified channel(s) to avoid being called every time a bit of IO arrives. The library will execute the callback whenever the specified number of bytes becomes available. Any remaining buffered data will be "flushed" upon call to deregister the respective channel
PMIX_IOF_BUFFERING_TIME
"Pmix.iof.btime"
(uint32_t) max time in seconds to buffer IO before delivering it. Used in conjunction with buffering size, this prevents IO from being held indefinitely while waiting for another payload to arrive
PMIX_IOF_COMPLETE
"Pmix.iof.cmp"
(bool) indicates whether or not the specified IO channel has been closed by the source
PMIX_IOF_PUSH_STDIN
"Pmix.iof.stdin"
(pmixproc_t*) Used by a tool to request that the PMIx library collect the tool's stdin and forward it to the indicated proc - the PMIX_RANK_WILDCARD value indicates that all procs in the provided nspace are to receive the data
```

#### Error Codes
```
PMIX_ERR_NOT_SUPPORTED
```

### References

1. https://github.com/pmix/RFCs/blob/master/RFC0025.md
2. https://pmix.org/pmix-standard/input-output-forwarding-for-tools/

