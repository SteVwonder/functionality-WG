## Brief Description

This use case is an attempt to distill out the features/extensions requested in the RFCs that are related to debugging. We have identified parts of PR23 (Co-located process launch for debuggers), RFC0010 (MPIR-like query), RFC0002 (event pub/sub), and RFC0022 (Environmental Parameter Directives for Applications and Launchers) under this category.

## Terminology

### Tools vs Debuggers

 - Tool: processes designed to monitor, record, analyze, or control the execution of another process.  Typically used for the purposes of profiling and debugging.
	 - First-Party Tool: run within the address space of the application process
	 - Third-Party Tool: run within its own process
	 - Hybrid Tool: runs in both its own process and the address space of the application process
- Debugger: third-party tool that inspects and controls an application process's execution using system-level debug APIs (e.g., `ptrace`)

### Parallel Launching Methods
A *starter* program is a program responsible for launching a parallel runtime, such as MPI.  PMIx supports two primary methods for launching parallel applications under tools and debuggers: indirect and direct. In the indirect launching method, the tool is attached to the starter.  In the direct launching method, the tool takes the place of the starter.  PMIx also supports attaching to already running programs via the *Process Acquisition* interfaces.

### Process Synchronization
Process Synchronization is the technique tools use to start the processes of a parallel application such that the tools can still attach to the process early in it's lifetime.  Said another away, the tool must be able to start the application processes without them "running away" from the tool.  In the case of MPI, this means stopping the applications processes before they return from `MPI_Init`.

### Process Acquisition<a name="process-acq"></a>

Process Acquisition is technique tools use to locate all of the processes, local and remote, of a given parallel application.  This typically boils down to collecting for every process in the parallel application:
- the hostname or IP of the machine running the process
- the executable name
- the process ID


## Use Case Details
### 1.  Direct-Launch Debugger Tool

PMIx can support the tool itself using the PMIx spawn options to control the app’s startup, including directing the RM/application as to when to block and wait for tool attachment, or stipulating that an interceptor library be preloaded. However, this means that the user is restricted to whatever command line options the tool vendor has provided for operations such as process placement and binding, which places a significant burden on the tool vendor. An example might look like the following:

`dbgr -n 3 ./myapp`

Assuming it is supported, co-launch of debugger daemons in this use-case is supported by adding a `pmix_app_t` to the `PMIx_Spawn command`, indicating that the resulting processes are debugger daemons by setting the `PMIX_DEBUGGER_DAEMONS` attribute.

![test](https://i.imgur.com/oQTqRbQ.jpg)

#### Interfaces
```
PMIx_tool_init
PMIx_Register_event_handler
PMIx_Query_info
PMIx_Spawn
PMIx_Get
PMIx_Notify_event
```
#### Attributes/Directives
```
PMIX_QUERY_SPAWN_SUPPORT
PMIX_QUERY_DEBUG_SUPPORT
PMIX_DEBUG_STOP_IN_INIT
PMIX_FWD_STDOUT
PMIX_FWD_STDERR
PMIX_NOTIFY_COMPLETION
PMIX_SETUP_APP_ENVARS
PMIX_DEBUGGER_DAEMONS
PMIX_DEBUG_JOB
PMIX_DEBUG_WAITING_FOR_NOTIFY
PMIX_QUERY_LOCAL_PROC_TABLE
PMIX_ERR_DEBUGGER_RELEASE
```

### 2. Indirect-Launch Debugger Tool

Executing a program under a tool using an intermediate launcher such as mpiexec can also be made possible. This requires some degree of coordination between the tool and the launcher. Ultimately, it is the launcher that is going to launch the application, and the tool must somehow inform it (and the application) that this is being done in a debug session so that the application knows to “block” until the tool attaches to it.

In this operational mode, the user invokes a tool (typically on a non-compute, or “head”, node) that in turn uses mpiexec to launch their application – a typical command line might look like the following:
`dbgr -dbgoption mpiexec -n 32 ./myapp`

![enter image description here](https://i.imgur.com/TTWJbvl.jpg)

#### Interfaces
```
PMIx_tool_init
PMIx_Register_event_handler
PMIx_Spawn
PMIx_Notify_event
PMIx_tool_connect_to_server
PMIx_Query_info
PMIx_Get
```
#### Attributes/Directives
```
PMIX_LAUNCHER_READY
PMIX_LAUNCHER_COMPLETE
PMIX_SPAWN_TOOL
PMIX_FWD_STDOUT
PMIX_FWD_STDERR
PMIX_SETUP_APP_ENVARS
PMIX_LAUNCHER_READY
PMIX_LAUNCHER_DIRECTIVE
PMIX_DEBUG_STOP_IN_INIT
PMIX_LAUNCH_COMPLETE
PMIX_QUERY_PROC_TABLE
PMIX_DEBUGGER_DAEMONS
PMIX_DEBUG_JOB
PMIX_FWD_STDOUT
PMIX_FWD_STDERR
PMIX_NOTIFY_COMPLETION
PMIX_DEBUG_WAITING_FOR_NOTIFY
PMIX_SETUP_APP_ENVARS
PMIX_DEBUG_JOB
PMIX_QUERY_LOCAL_PROC_TABLE
PMIX_ERR_DEBUGGER_RELEASE
```

### 3. Attaching to a Running Job

PMIx supports attaching to an already running parallel job in two ways.  In the first way, the main process of a tool calls `PMIx_Query_info` with the `PMIX_QUERY_PROC_TABLE` attribute.  This returns an array of structs containing the information required for [*process acquisition*](#process-acq).  This includes remote hostnames, executable names, and process IDs.  In the second way, every tool daemon calls `PMIx_Query_info` with the `PMIX_QUERY_LOCAL_PROC_TABLE` attribute.  This returns a similar array of structs but only for processes on the same node.

An example of this use-case may look like the following:
```
mpiexec -n32 ./myApp
dbgr attach $!
```

![Process Acquisition](https://i.imgur.com/jTBIV8w.jpg)

#### Interfaces
```
PMIx_tool_init
PMIx_Register_event_handler
PMIx_Query_info
PMIx_Spawn
```

#### Attributes/Directives
```
PMIX_QUERY_ALL_NAMESPACES
PMIX_QUERY_PROC_TABLE
PMIX_DEBUGGER_DAEMONS
PMIX_DEBUG_JOB
PMIX_FWD_STDOUT
PMIX_FWD_STDERR
PMIX_NOTIFY_COMPLETION
PMIX_SETUP_APP_ENVARS
```

### 4. Tool Interaction with RM

Tools can benefit from a mechanism by which they may interact with a local PMIx server that has opted to accept such connections along with support for tool connections to system-level PMIx servers, and a logging feature. To add support for tool connections to a specified system-level, PMIx server environments could choose to launch a set of PMIx servers to support a given allocation - these servers will (if so instructed) provide a tool rendezvous point that is tagged with their pid and typically placed in an allocation-specific temporary directory to allow for possible multi-tenancy scenarios. Supporting such operations requires that a system-level PMIx connection be provided which is not associated with a specific user or allocation. A new key has been added to direct the PMIx server to expose a rendezvous point specifically for this purpose.

#### Interfaces
```
PMIx_Query_info_nb
PMIx_server_init
PMIx_Register_event_handler
PMIx_Deregister_event_handler
PMIx_Notify_event
```

#### Job-specific events
`PMIX_EVENT_JOB_LEVEL       /* debugger attached, process failure */`

 #### Environment events
`PMIX_EVENT_ENVIRO_LEVEL  /*ECC errors, temperature excursions */`

#### Errors detected by clients/peers
`Network fabric manager detects data corruption`

### 5. Environmental Parameter Directives for Applications and Launchers

It is sometimes desirable or required that standard environmental variables (e.g., `PATH`, `LD_LIBRARY_PATH`, `LD_PRELOAD`) be modified prior to executing an application binary or a starter such as mpiexec - this is particularly true when tools/debuggers are used to start the application. This RFC proposes the definition of a new PMIx structure (pmix_envar_t) and associated attributes for specifying such operations.

#### Interfaces
```
PMIx_Spawn
```
```
typedef struct {
    char *envar;
    char *value;
    char separator;
} pmix_envar_t;
```
#### Attributes/Directives
```
PMIX_SET_ENVAR
"pmix.envar.set"
(pmix_envar_t*) set the envar to the given value, overwriting any pre-existing one
PMIX_ADD_ENVAR
"pmix.envar.add" 
(pmix_envar_t*) add envar, but do not overwrite any existing one
PMIX_UNSET_ENVAR
"Pmix.envar.unset"
(char*) unset the envar, if present
PMIX_PREPEND_ENVAR
"Pmix.envar.prepnd"
(pmix_envar_t*) prepend the given value to the specified envar using the separator character, creating the envar if it doesn't already exist
PMIX_APPEND_ENVAR
"pmix.envar.appnd"
(pmix_envar_t*) append the given value to the specified envar using the separator character, creating the envar if it doesn't already exist
```
Resource managers and launchers must scan for relevant directives, modifying environmental parameters as directed. Directives are to be processed in the order in which they were given, starting with job-level directives (applied to each app) followed by app-level directives.

## References
1. https://github.com/pmix/RFCs/pull/23
2. https://github.com/pmix/RFCs/blob/master/RFC0010.md
3. https://github.com/pmix/RFCs/blob/master/RFC0002.md
4. https://github.com/pmix/RFCs/blob/master/RFC0022.md
5. https://pmix.org/support/how-to/example-indirect-launch-debugger-tool/
6. https://pmix.org/support/how-to/example-direct-launch-debugger-tool/
7. https://github.com/openpmix/openpmix/blob/6a8cc1ca0523b531b20a9a0f7bf7b27c9b5c6023/examples/debugger.c
