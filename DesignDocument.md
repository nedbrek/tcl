# Introduction #



**Google Summer of Code 2011**

Tcl/tk community

# Design Document #


Project:
**Micro-benchmarking extension: access to CPU performance counters**

Developer student: Saurabh Kumar

Project Mentor: Edward Brekelbaum



# Details #

**Project Details**

The goal of this project is to design and implement a Tcl extension with commands to interact with the CPU's hardware counters. We first plan to code an extension that works under Linux.
The proposed Tcl extension will be based on Tcl C API.
There exist tools like the Linux performance counter subsystem (Perf) that provide access to the hardware counters. Perf provides rich abstractions over the hardware counters' capabilities. It provides per task, per CPU and per-workload counters, counter groups, and it provides sampling capabilities on top of those and more. It also provides abstraction for 'software events' - such as minor/major page faults, task migrations, task context-switches and tracepoints.

There are several ways one could use the Perf tool. For example, The perf tool can be used to get access to the hardware counters by using the perf syscalls ( sys\_perf\_counter\_open syscall etc.).

There exist an opensource software package Performance Application Programming Interface
(PAPI http://icl.cs.utk.edu/papi/) that aims to provide the tool designer and application engineer with a consistent interface and methodology for use of the performance counter hardware found in most major microprocessors.

It provides a nice API which makes the access of hardware counters through C/C++ code quite easy.
It appears to be a wrapper around Perf to allow some additional platforms.

We plan to use this software package to get access to the hardware counters.

## Implementation details ##


**Programming Language to be used:**  C++

## Basic Structure of the code: ##
We plan to create an ensemble bound to a namespace, which consists of a collection of subcommands. The following code fragment illustrates what we plan to implement:

```
int perft_Init(Tcl_Interp *interp)
{
 // create namespace for the commands
 Tcl_Namespace *ns = Tcl_CreateNamespace(interp, "perft", NULL, NULL);

 // tell Tcl to grab all subcommands on import
 Tcl_Export(interp, ns, "*", 0);

 // create the subcommands

 // Initialises the perf tool
 Tcl_CreateObjCommand(interp, "perft::init", Init_Cmd, NULL, NULL);

 // Runs the tool on the file name given as command line argument
 Tcl_CreateObjCommand(interp, "perft::run_file", Run_Cmd_File, NULL, NULL);

 // Runs the tool on the tcl script given as command line argument
 Tcl_CreateObjCommand(interp, "perft::run_script", Run_Cmd_Script, NULL, NULL);

 // create the ensemble
 Tcl_CreateEnsemble(interp, "perft", ns, 0);

 // success
 return TCL_OK;
}

```

This code creates a “perft” namespace. We plan to implement the following three sub-commands:
1) perft::init
2) perft::run\_file
2) perft::run\_script







## The perft::init sub-command: ##

The init sub-command  is meant for initializing the PAPI library. It takes a list of event names as argument. This list should contain all the events which should be under investigation during the run commands.
e.g. if you want to count the following events:
1) L1 data cache misses
2) Total cycles
3) Instructions issued
4) Floating point operations

the following list should be passed as an argument to the init subcommand

`[list "PAPI_L1_DCM" "PAPI_TOT_CYC" "PAPI_TOT_IIS" "PAPI_FP_OPS"]`

So  we could write like

`set events_list [list "PAPI_L1_DCM" "PAPI_TOT_CYC" "PAPI_TOT_IIS" "PAPI_FP_OPS"];`

and then init command can be called as

perft::init $events\_list;

After initializing the PAPI library, init subcommand creates a PAPI eventset for the given event\_list.

So the code would be somewhat like

```
static int Init_Cmd(ClientData cdata, Tcl_Interp *interp, int objc, Tcl_Obj *const objv[]) {

// initaizing library
if (PAPI_library_init(PAPI_VER_CURRENT) != PAPI_VER_CURRENT)
 {
    printf("Error initializing the PAPI Library");
 exit(1);
 }
.
.
.
//creating event set
if (PAPI_create_eventset(&event_set) != PAPI_OK) {
    printf("Error in subroutine PAPIF_create_eventset");
   exit(1);
}
.
.
.
for all events
do
{
PAPI_add_event(event_set, event_code);
}
.
.
.

```


Apart from this the init subcommand will also complete certain other operations that should be completed before calling the run commands e.g. allocating sufficient memory for storing results etc.

After running the init subcommand tcl interpreter will respond like

% perft::init $events\_list;
Hardware Counters Initialized

in case the initialization is successful or will print appropriate error message in case of some error in initialization.


## The perft::available\_events sub-command: ##



This command is useful in selecting proper PAPI event names for different hardware events.
As a return value, it returns a Tcl object storing a string list containing the PAPI preset events with little description about each of them.

The structure of the list is:
[event\_name1 event\_description1 event\_name2 event\_description2.......](list.md)

The C++ code for this command uses PAPI\_enum\_event() and PAPI\_get\_event\_info() functions.

The command will work as illustrated below in the tcl interpreter:

```


%set all_events [perft::available_events]
```

On a successful run it will return the recorded data in the following format

```

PAPI_L1_DCM {Level 1 data cache misses} PAPI_L1_ICM {Level 1 instruction cache misses} PAPI_L2_DCM {Level 2 data cache misses} ......


```

In case of any error it will output appropriate error message and exit.


## The perft::select\_events sub-command: ##



This command is meant to make the event selection easy by providing a GUI using tk. As soon as this command is run, a tk widget is created which allows users to choose among the possible hardware events comfortably.

Once the user presses "done" button, it also calls perft::init command with the selected event names' list as input argument.


The command will work as illustrated below in the tcl interpreter:

```


%perft::select_events
```

On a successful run it will show a widget like this

![http://wiki.tcl.googlecode.com/hg/widget.png](http://wiki.tcl.googlecode.com/hg/widget.png)


If everything goes fine, hardware counters are initialised with the selected event set.
```


success!!
Hardware counters initialized
```


In case of any error appropriate error message is displayed and the program is exited.


## The perft::run\_file sub-command: ##


This command should follow the successful call of perft::init subcommand. After the initialization of PAPI library and the PAPI event\_set, the run\_file subcommand uses PAPI\_start() function to start counting hardware events in an event set. It then calls appropriate functions to run the tcl script written in the the file name given as the command line argument.

After the script is over it calls PAPI\_stop() to stop counting hardware events in an event set and collect the results from hardware counters. Moreover it also resets the event set and other relevant things so that perft::init can be called again later.

As a return value, it returns a Tcl object storing a string list containing the data recorded by the hardware counters.

The following code fragment illustrates the functioning.


The C++ code for this command will look like

```
static int Run_file_Cmd(ClientData cdata, Tcl_Interp *interp, int objc, Tcl_Obj *const objv[])  {    
.
.
.
// start the counters
 if(PAPI_start(event_set)!= PAPI_OK) {
printf("Abort After PAPIF_start: ");
    exit(1);
   }
   
   // run the script
   Tcl_EvalFile(interp,file_name);


//stop counting

     if (PAPI_stop(event_set, values)!= PAPI_OK) {
     printf("Abort After PAPIF_stop: ");
         exit(1);
     }
.
.
.
   PAPI_cleanup_eventset(event_set);
..
   PAPI_destroy_eventset(&event_set);
..
   event_set=PAPI_NULL;
..

// return the result in appropriate format
.
.
.
.
   
}

```

The command will work as illustrated below in the tcl interpreter (assuming perft::init has been already called):

%perft::run\_file file\_name

On a successful run it will return the recorded data in the following format e.g.

```
%set events_list [list "PAPI_L1_DCM" "PAPI_TOT_CYC" "PAPI_TOT_IIS" "PAPI_FP_OPS"];
.
.
.
%perft::init $events_list;
Hardware Counters Initialized
.
.
.

%perft::run_file file_name;
.
.
PAPI_L1_DCM 3602 | PAPI_TOT_CYC 961924 | PAPI_TOT_IIS 963808 | PAPI_FP_OPS 1604
```

This command can also be used to accumulate the results to the previous counter values using the flag '-a'. This is illustrated below:

```
%set events_list [list "PAPI_L1_DCM" "PAPI_TOT_CYC" "PAPI_TOT_IIS" "PAPI_FP_OPS"];
.
.
.
%perft::init $events_list;
Hardware Counters Initialized
.
.
.

%perft::run_file file_name;
.
.
PAPI_L1_DCM 3602 | PAPI_TOT_CYC 961924 | PAPI_TOT_IIS 963808 | PAPI_FP_OPS 1604

%perft::run_file file_name -a;
.
.
PAPI_L1_DCM 7000 | PAPI_TOT_CYC 1763434 | PAPI_TOT_IIS 1898743 | PAPI_FP_OPS 3005

%perft::run_file file_name;
.
.
PAPI_L1_DCM 3013 | PAPI_TOT_CYC 901564 | PAPI_TOT_IIS 894545 | PAPI_FP_OPS 1567
```


## The perft::run\_script sub-command: ##



This command is quite similar to the run\_file command, the only difference being that the command line argument is a string variable containing the tcl script to be executed.
As a return value, it returns a Tcl object storing a string list containing the data recorded by the hardware counters.

The following code fragment illustrates the functioning.


The C++ code for this command will look like

```
static int Run_script_Cmd(ClientData cdata, Tcl_Interp *interp, int objc, Tcl_Obj *const objv[])  {    
.
.
.
// start the counters
 if(PAPI_start(event_set)!= PAPI_OK) {
printf("Abort After PAPIF_start: ");
    exit(1);
   }
   
   // run the script
   Tcl_EvalObjEx(interp, string_argument, flags);


//stop counting

     if (PAPI_stop(event_set, values)!= PAPI_OK) {
     printf("Abort After PAPIF_stop: ");
         exit(1);
     }
.
.
.
PAPI_cleanup_eventset(event_set);
   PAPI_destroy_eventset(&event_set);
   event_set=PAPI_NULL;

// return the result in appropriate format
.
.
.
.
   
}

```
The command will work as illustrated below in the tcl interpreter (assuming perft::init has been already called):

%perft::run\_script $script\_string

On a successful run it will return the recorded data in the following format e.g.

```
%set events_list [list "PAPI_L1_DCM" "PAPI_TOT_CYC" "PAPI_TOT_IIS" "PAPI_FP_OPS"];
.
.
.
%perft::init $events_list;
Hardware Counters Initialized
.
.
%set script_string {puts [expr 2+3]};
.

%perft::run_script $script_string;
5
PAPI_L1_DCM 394 | PAPI_TOT_CYC 54897 | PAPI_TOT_IIS 46274 | PAPI_FP_OPS 27

```

This command can also be used to accumulate the results to the previous counter values using the flag '-a'. This is illustrated below:

```
%set events_list [list "PAPI_L1_DCM" "PAPI_TOT_CYC" "PAPI_TOT_IIS" "PAPI_FP_OPS"];
.
.
.
%perft::init $events_list;
Hardware Counters Initialized
.
.
.

%perft::run_file file_name;
.
.
5
PAPI_L1_DCM 394 | PAPI_TOT_CYC 54897 | PAPI_TOT_IIS 46274 | PAPI_FP_OPS 27
.
%perft::run_file file_name -a;
.
.
5
PAPI_L1_DCM 714 | PAPI_TOT_CYC 94236 | PAPI_TOT_IIS 76986 | PAPI_FP_OPS 40
%perft::run_file file_name;
.
.
PAPI_L1_DCM 301 | PAPI_TOT_CYC 45764 | PAPI_TOT_IIS 43545 | PAPI_FP_OPS 25
```


**Recent Additions**

1) perft:counters command can be used to get the number of hardware counters.

2) To get more precise results "cpuid" assembly instruction has been used to enforce serialization.

3) The support for windows has been provided. Though it is not based on PAPI but uses perfinsp APIS http://perfinsp.sourceforge.net/. The high level interface is same for both the platforms but the namespaces for events differ.