## Brief Description

Hybrid applications (i.e., applications that utilize more than one programming model, such as an MPI application that also uses OpenMP or PGAS) are growing in popularity, especially as chips with increasingly large numbers of cores and processors proliferate. Unfortunately, the various models currently operate under the assumption that they alone control execution. This leads to conflicts in hybrid applications. Deadlock of parallel applications can occur when one model prevents the other from making progress due to lack of coordination between the multiple programming models [\[1\]](http://mvapich.cse.ohio-state.edu/static/media/talks/slide/SC15-PGAS-final.pdf).  Sub-optimal performance can also occur due to uncoordinated division of hardware resources between the programming models [2] [3].  This use-case offers potential solutions to the problem by providing a pathway for programming models to coordinate their actions.

## Use Case Details

### Identifying Active Programming Models

The current state-of-the-practice for programming models to detect one another is via set environment variables.  For example, OpenMP looks for environment variables to indicate that MPI is active.  Unfortunately, this technique is not completely reliable as environment variables change over time and with new software versions.  Also, the fact that an environment variable is present doesn't guarantee that a particular programming is in active use since Resource Managers routinely set environment variables "just in case" the application needs them. PMIx provides a reliable mechanism by which each library can determine that another library is in operation.

\When initializing PMIx, programming models can register themselves, including their name, version, and threading model.  This information is then cached locally and can then be read asynchronously by other programming models using PMIx's Event Notification system (see next section for more details).

This initialization mechanism also allows libraries to share knowledge of each other's resources and intended resource utilization. For example, if OpenMP knows which hardware threads that MPI is using it could potentially avoid processor and cache contention.

#### Interface
`PMIx_Init`

#### Attributes
```C
#define PMIX_PROGRAMMING_MODEL              "pmix.pgm.model"        // (char*) programming model being initialized (e.g., "MPI" or "OpenMP")
#define PMIX_MODEL_LIBRARY_NAME             "pmix.mdl.name"         // (char*) programming model implementation ID (e.g., "OpenMPI" or "MPICH")
#define PMIX_MODEL_LIBRARY_VERSION          "pmix.mdl.vrs"          // (char*) programming model version string (e.g., "2.1.1")
#define PMIX_THREADING_MODEL                "pmix.threads"          // (char*) threading model used (e.g., "pthreads")
#define PMIX_MODEL_NUM_THREADS              "pmix.mdl.nthrds"           // (uint64_t) number of active threads being used by the model
#define PMIX_MODEL_NUM_CPUS                 "pmix.mdl.ncpu"             // (uint64_t) number of cpus being used by the model
#define PMIX_MODEL_CPU_TYPE                 "pmix.mdl.cputype"          // (char*) granularity - "hwthread", "core", etc.
#define PMIX_MODEL_PHASE_NAME               "pmix.mdl.phase"            // (char*) user-assigned name for a phase in the application
                                                                        //         execution - e.g., "cfd reduction"
#define PMIX_MODEL_PHASE_TYPE               "pmix.mdl.ptype"            // (char*) type of phase being executed - e.g., "matrix multiply"
#define PMIX_MODEL_AFFINITY_POLICY          "pmix.mdl.tap"              // (char*) thread affinity policy - e.g.:
                                                                        //           "master" (thread co-located with master thread),
                                                                        //           "close" (thread located on cpu close to master thread)
                                                                        //           "spread" (threads load-balanced across available cpus)
```

#### Code Example
```C
pmix_proc_t myproc;
pmix_info_t *info;
PMIX_INFO_CREATE(info, 4);
PMIX_INFO_LOAD(&info[0], PMIX_PROGRAMMING_MODEL, "MPI", PMIX_STRING);
PMIX_INFO_LOAD(&info[1], PMIX_MODEL_LIBRARY_NAME, "FooMPI", PMIX_STRING);
PMIX_INFO_LOAD(&info[2], PMIX_MODEL_LIBRARY_VERSION, "1.0.0", PMIX_STRING);
PMIX_INFO_LOAD(&info[3], PMIX_THREADING_MODEL, "pthread", PMIX_STRING);
pmix_status_t rc = PMIx_Init(&myproc, info, 4);
PMIX_INFO_FREE(info, 4);
```


### Coordinating at Runtime

The PMIx Event Notification system provides a mechanism by which the resource manager can communicate system events to applications, thus providing applications with an opportunity to generate an appropriate response. Hybrid applications can leverage these events for cross-library coordination.

Programming models can access the information provided by other programming models during their initialization using the event notification system.  In this case, programming models should register a callback for the `PMIX_MODEL_DECLARED` event.

Programming models can also use the PMIx event notification system to communicate dynamic information, such as entering  a new application phase (`PMIX_MODEL_PHASE_NAME`) or a change in resources used (`PMIX_MODEL_RESOURCES`).  This dynamic information can be broadcast to other programming models using the `PMIx_Notify_event` function.  Other programming models can register callback functions to run when these events occur (i.e., callback functions) using `PMIx_Register_event_handler`.

#### Interfaces
```
PMIx_Notify_event
PMIx_Register_event_handler
```

#### Status Codes
```C
#define PMIX_MODEL_DECLARED                 (PMIX_ERR_OP_BASE - 17)
#define PMIX_MODEL_RESOURCES                (PMIX_ERR_OP_BASE - 21)     // model resource usage has changed
#define PMIX_OPENMP_PARALLEL_ENTERED        (PMIX_ERR_OP_BASE - 22)     // an OpenMP parallel region has been entered
#define PMIX_OPENMP_PARALLEL_EXITED         (PMIX_ERR_OP_BASE - 23)     // an OpenMP parallel region has completed
```

#### Code Example
Registering a callback to run when another programming model initializes:
```c
static void model_declared_cb(size_t evhdlr_registration_id, pmix_status_t status, const pmix_proc_t *source, pmix_info_t info[], size_t ninfo, pmix_info_t results[], size_t nresults, pmix_event_notification_cbfunc_fn_t cbfunc, void *cbdata)
{
    for (n = 0; n < ninfo; n++)  {
        if  (PMIX_CHECK_KEY(info[n], PMIX_PROGRAMMING_MODEL) &&
             strcmp(info[n].value.data.string, "MPI") == 0) {
            /* ignore our own declaration */
            break;
        } else {
            /* do whatever we want/need MPI to do when another model registers */
        }
    }
    if (NULL != cbfunc) {
        /* tell the event handler state machine that we are only a partial step and to continue the event handler chain */
        cbfunc(PMIX_EVENT_PARTIAL_ACTION_TAKEN, NULL, 0, NULL, NULL, cbdata);
    }
}

pmix_status_t code = PMIX_MODEL_DECLARED;
rc = PMIx_Register_event_handler(&code, 1, NULL, 0, model_declared_cb, NULL, NULL);
```

Notifying an event:
```c
// TODO
PMIX_INFO_CREATE(info, 2);
PMIX_INFO_LOAD(&info[0], PMIX_EVENT_NON_DEFAULT, false, PMIX_BOOL);
PMIX_INFO_LOAD(&info[1], PMIX_EVENT_CUSTOM_RANGE, TODO, TODO);
PMIx_Notify_event(PMIX_OPENMP_PARALLEL_ENTERED, TODO, TODO, NULL, 0, NULL, NULL);
PMIX_INFO_FREE(info, 2);
```

### Coordination at Runtime with Multiple Event Handlers

Coordinating with a threading library such as OpenMP creates the need for separate event handlers for threads of the same process.  For example in an MPI+OpenMP hybrid application, the MPI thread and the main OpenMP thread may both want to be notified anytime an OpenMP worker thread enters a parallel region.  This requiring support for multiple threads to potentially register different event handlers against the same status code.

Multiple event handlers registered against the same event are processed in a chain-like manner based on the order in which they were registered, as modified by directive. Registrations against specific event codes are processed first, followed by registrations against multiple event codes and then any default registrations. At each point in the chain, an event handler is called by the PMIx progress thread and given a function to call when that handler has completed its operation. The handler callback notifies PMIx that the handler is done, returning a status code to indicate the result of its work. The results are appended to the array of prior results, with the returned values combined into an array within a single pmix_info_t as follows:
1. array[0]: the event handler name provided at registration (may be an empty field if a string name was not given) will be in the key, with the pmix_status_t value returned by the handler
2. array[*]: the array of results returned by the handler, if any.

The current PMIx standard does not actually specify a default ordering for event handlers as they are being registered. However, it does include an inherent ordering for invocation. Specifically, PMIx stipulates that handlers be called in the following categorical order:
1. single status event handlers - i.e., handlers that were registered against a single specific status.
2. multi status event handlers - those registered against more than one specific status
3. default event handlers - those registered against no specific status

#### Interfaces
```
PMIx_Register_event_handler
```

#### Attributes 
```
PMIX_EVENT_HDLR_NAME
PMIX_EVENT_JOB_LEVEL
PMIX_EVENT_ENVIRO_LEVEL
PMIX_EVENT_ORDER_PREPEND
PMIX_EVENT_HDLR_FIRST
PMIX_EVENT_HDLR_LAST
PMIX_EVENT_HDLR_FIRST_IN_CATEGORY
PMIX_EVENT_HDLR_LAST_IN_CATEGORY
PMIX_EVENT_HDLR_BEFORE
PMIX_EVENT_HDLR_AFTER
PMIX_EVENT_HDLR_APPEND
PMIX_RANGE_PROC_LOCAL
```

#### Status Codes
```
PMIX_EVENT_NO_ACTION_TAKEN /*indicates that the handler didn't take any action on this event*/
PMIX_EVENT_PARTIAL_ACTION_TAKEN /*the handler took some action, but did not fully resolve the situation*/
PMIX_EVENT_ACTION_DEFERRED /*the handler has queued actions to be taken later*/
PMIX_EVENT_ACTION_COMPLETE /*the handler has completely resolved the event and no further handlers should be called*/
PMIX_EVENT_NO_TERMINATION /* indicates that the handler has satisfactorily handled the event and believes termination of the application is not required*/
PMIX_EVENT_WANT_TERMINATION /* indicates that the handler has determined that the application should be terminated*/
```

#### Code Example
From the OpenMP master thread:
```c
static void parallel_region_entered_cb(...)
{
    /* do whatever we want/need MPI to do on entering a parallel region */
    if (NULL != cbfunc) {
        /* tell the event handler state machine that we are only a partial step and to continue the event handler chain */
        cbfunc(PMIX_EVENT_PARTIAL_ACTION_TAKEN, NULL, 0, NULL, NULL, cbdata);
    }
}
pmix_status_t code = PMIX_OPENMP_PARALLEL_ENTERED;
PMIX_INFO_CREATE(info, 2);
PMIX_INFO_LOAD(&info[0], PMIX_EVENT_HDLR_NAME, "OpenMP-Master", PMIX_STRING);
PMIX_INFO_LOAD(&info[1], PMIX_EVENT_HDLR_FIRST, true, PMIX_BOOL);
rc = PMIx_Register_event_handler(&code, 1, info, 1, parallel_region_entered_cb, NULL, NULL);
PMIX_INFO_FREE(info, 2);
```

From the MPI thread:
```c
static void parallel_region_entered_cb(...)
{
    /* do whatever we want/need MPI to do on entering a parallel region */
    if (NULL != cbfunc) {
        /* tell the event handler state machine that we are the last step */
        cbfunc(PMIX_EVENT_ACTION_COMPLETE, NULL, 0, NULL, NULL, cbdata);
    }
}
pmix_status_t code = PMIX_OPENMP_PARALLEL_ENTERED;
PMIX_INFO_CREATE(info, 2);
PMIX_INFO_LOAD(&info[0], PMIX_EVENT_HDLR_NAME, "MPI-Thread", PMIX_STRING);
PMIX_INFO_LOAD(&info[1], PMIX_EVENT_HDLR_AFTER, "OpenMP-Master", PMIX_STRING);
rc = PMIx_Register_event_handler(&code, 1, info, 1, parallel_region_entered_cb, NULL, NULL);
PMIX_INFO_FREE(info, 2);
```

### References

 1. Supporting Hybrid MPI+PGAS Programming Models through Unified Communication Runtime: An MVAPICH2-X Approach. Khaled Hamidouche, Jian Lin, Mingzhe Li, Jie Zhang and D.K.Panda. 4th MVAPICH User's Group (MUG). 2016. http://mvapich.cse.ohio-state.edu/static/media/talks/slide/SC15-PGAS-final.pdf
 2. Improving Support of MPI+OpenMP Applications. Geoffroy R. Vallée and David Bernhold. Proceedings of the 25th European MPI Users’ Group Meeting, EuroMPI 2018. https://eurompi2018.bsc.es/sites/default/files/uploaded/eurompi2018-paper-vallee.pdf
 3. https://github.com/OMPI-X/MOC
 4. https://github.com/pmix/RFCs/blob/master/RFC0017.md
 5. https://github.com/pmix/RFCs/blob/master/RFC0002.md
 6. https://github.com/pmix/RFCs/blob/master/RFC0018.md
