Copying over some of @jdelsign's feedback from an off-thread conversation:

- If we follow OpenMP's lead, I think "tool" is a more generic term, which is
  subdivided into first-party and third-party tools. A first-party tool runs
  within the address space of the application process, whereas a third-party
  tool runs in its own process. A debugger is a third-party tool that controls
  the application process normally using system-level debug APIs (think
  ptrace(), but not necessarily). I can also imagine hybrid tools, where part of
  the tool runs in its own process and part runs within the address space of the
  application process. For example, TotalView is a third-party tool that can
  also inject memory debugging agents into the application.
- What does a debugger user need to do? In other words, what does it look like
  from a debugger user's perspective. Currently, w/ MPIR, a user will start the
  debugger on the MPI starter process or attach to the MPI starter process.
  - This is documented above as the "indirect launch" use-case
- At one point a while ago, Ralph and I thought one scenario would be that the
  debugger itself might take the place of the starter process. I'm not sure if
  that's still under consideration.
  - This is documented above as the "direct launch" use-case
- What does the debugger need to do to get the MPI process list? Basically, it's
  the stuff (host, exec, pid) that's in the MPIR_proctable[].
  - TIL that this is known as "Process Acquisition" in the MPIR spec
  - This information is contained in the `pmix_proc_info_t` struct. This can be
    obtained by making a call to `PMIx_Query_info_nb` with the
    `PMIX_QUERY_PROC_TABLE` attribute, which will return an array of those
    structs.  You can also use `PMIX_QUERY_LOCAL_PROC_TABLE` attribute to get
    the information on only processes in the same job on the same node.
- How does the tool start up on the job and prevent the job from "running away"
  before the tool has a chance to do its thing (attach or whatnot)?
  - TIL that this is known as "Process Synchronization" in the MPIR spec
  - When calling `PMIx_Spawn` to launch the job's processes, you can use the
    `PMIX_DEBUG_STOP_ON_EXEC`, the `PMIX_DEBUG_STOP_IN_INIT`, or the
    `PMIX_DEBUG_WAIT_FOR_NOTIFY` attributes.
- What does the debugger need to do start the debugger agents? I think there are
  a couple of sub-cases here: (a) the debugger does it directly itself, and (b)
  the debugger uses an external package. For TotalView, what I'd like to see is
  MRNet tree instantiation support layered on top of the PMIx support to
  potentially take the place of using ssh/rsh.
    - A) The debugger can use `PMIx_Spawn` to co-locate debugger agents with the
    job processes.
    - B) ???
- Is there any comms/wire-up support in PMIx? The debugger client needs to
  communicate with the debugger agents. In the case of MRNet, it manages its own
  sockets, but it would be nice to know if PMIx can help out in this respect.
- What is the process's rank in the job?
- What is the event delivery mechanism for (new) processes launch by the job?
- What is the name of the MQD DLL for the process?

One other use-case from the "Debuggers and Tools" presentation from @rhc54: attaching to an already running job

Might be good to start with a paragraph that summarizes the use-case and
provides terminology (define tools vs debuggers, different forms of launching,
what is attaching, what is process acquisition).

Add cospawn

We should reference this file: https://github.com/openpmix/openpmix/blob/6a8cc1ca0523b531b20a9a0f7bf7b27c9b5c6023/examples/debugger.c

What does the server-side need to support?
