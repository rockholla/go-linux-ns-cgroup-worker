# Linux Worker Service using Namespaces/CGroups Implemented in Golang

This is an experiement to build a job worker service that provides an API to run arbitrary Linux processes.

There will be 3 components to make up this service:

1. A Golang library to start, stop, and query the status/output of a worker job
2. A gRPC API to expose the functionality of this library over a network to start, stop, query status and stream output of a job, including authentication via mTLS and authorization scheme
3. A CLI for users to interact with the API to start, stop, query status/stream output of a job

> :pencil: Note that code and command samples here are meant to be approximations of the final implementation. Slight variations may exist in what is actually implemented in code from what you see in this design.

## Library

This will be the lowest-level part of the service stack, with the following goals and outcomes:

* providing an interface for a linux command/process to be executed on behalf of a user/owner
* interfacing with the linux OS internals/kernel via [`sys/unix`](https://pkg.go.dev/golang.org/x/sys/unix) to:
  * Add process isolation for a worker process in the following ways:
    * **PID**: no other processes executed by the worker service can see another worker process
    * **mounts**: dedicate file system space for the process
    * **networking**: isolate network interface and connectivity out of the linux machine, but with no routes to other processes/namespaces
  * Add resource usage control by assigning the spawned process to a cgroup, and limiting the amount of:
    * **CPU used by the process**: hard limit on the total CPU, **1 CPU**, that any single process could use from the host machine
    * **Memory used by the process**: hard limit on the total memory, **100 MB**, that any single process could use from the host machine
    * **Disk I/O rate used by the process**: hard limit on the read/write bytes/second available, **1 MB/s** to any single process
* Returning a unique service process ID for a successfully-triggered process/worker
* Ability to stop a running process by using this unique process ID
* Ability to get/stream the output/logs of the worker process by using the process ID, as well as get the final result exit code when available

### Library-Defined Process Store Interface

The library will define an interface for storing and tracking running processes, a process store, like:

```golang
// ProcessStore defines an interface for defining custom stores to track processes, ownership, status, output, etc.
type ProcessStore interface {
  NewProcess(workerID string, ownerID string) (err error)
  SetHostPID(workerID string, hostPID string) (err error)
  GetHostPID(workerID string) (hostPID string, err error)
  GetOutputWriters(workerID string) (stdout io.Writer, stderr io.Writer, err error)
  GetOwnerID(workerID string) (ownerID string, err error)
  GetStatus(workerID string) (done bool, exitCode int, pid string, err error)
  GetOutput(workerID string) (stdout io.Reader, stderr io.Reader, err error)
}
```

Our first implementation of this interface will be part of the libary and as simple as possible: a memory store tracking the owner, host pid, stdout, stderr, whether complete, and exit code for any requested worker.

This memory `ProcessStore` implementation will:

* Use `bytes.Buffer` for stdout/stderr, making it particularly easy to process as both an `io.Writer` and `io.Reader`. 
* Store everything in a map/hashtable, in memory, so it's easy for it to get at and write immediately to across our various parts that need to write to and read from it

Our library contracts are designed as follows:

```golang
type WorkerGroup struct {
  Store ProcessStore
}

// NewWorkerGroup will init and return a new worker group with a provider ProcessStore implementation
func NewWorkerGroup(store ProcessStore) *WorkerGroup

// Start will initiate a new worker/command on behalf of a user/owner
func (wg *WorkerGroup) Start(cmd []string, ownerID string) (workerID string, err error)

// Stop will take in a worker ID and the ID of the user requesting the stop, determine if a stop is allowed, and then do it
func (wg *WorkerGroup) Stop(workerID string, requesterID string) error

// GetStatus is a wrapper around the process store implementation to also manage authorization-related logic, e.g. does requester = owner
func (wg *WorkerGroup) GetStatus(workerID string, requesterID string) (done bool, exitCode int, err error)

// GetOutput is a wrapper around the process store implementation to also manage authorization-related logic, e.g. does requester = owner
func (wg *WorkerGroup) GetOutput(workerID string, requesterID string) (stdout io.Reader, stderr io.Reader, err error)
```

### Details of Linux Namespacing and Cgroup Execution

> :bulb: I don't now how we could use things like `syscall`/`sys/unix` `CLONE_INTO_CGROUP` or `SysProcAttr.CgroupFD`. I've never used them, and usage of them/knowledge of how to use them seems _very_ limited in my, well, also limited experience with how they are supposed to work. So, we'll stick to other paths here.

Namespace isolation while executing our user's command is achieved generally by setting [`Cloneflags`/`Unshareflags`](https://pkg.go.dev/syscall#pkg-constants) for the `SysProcAttr` on a standard [`os/exec.Cmd` object](https://pkg.go.dev/os/exec#Cmd). That's the simple part.

However, because of how Golang doesn't support certain ideas around `fork` natively, we'll need to wrap the user-specified command process in a wrapped execution of our own making to ensure that the namespace isolation and cgroup control actually work. Well do that by doing the following:

* Building a pre-built binary as an artifact that should be installed on a system using this library, e.g. `rockholla-go-linux-ns-cgroup-worker-fork` (we will make sure the build/install process and tooling does this automatically, for example if you're installing the built server binary, you'll get this one as well as a dependency)
* Making sure the library will call it as a pass-thru/wrapper command for the user-provided command
  * `rockholla-go-linux-ns-cgroup-worker-fork -- ping -c 2 google.com` as an example of the command the library will call via `exec.Command`
* This fork/wrapper will do the necessary per-process-level setup via the compiled Golang code of namespace isolation/cgroup limits for our needs before calling the user-provided command to be run w/in the isolated namespace/cgroup:
  * `cgroup` setup at `/sys/fs/cgroup/**/*` for our resource control needs of a single process
  * `CLONE_NEWNS`: we will need to set up a dedicated root filesystem for use by the process, and [`PivotRoot` to it](https://pkg.go.dev/golang.org/x/sys/unix#PivotRoot) (`pivot_root` system call), and set up some expected paths to common system/user binary paths, e.g. `/usr/bin` `/usr/local/bin` etc.
  * `CLONE_NEWPID`: we will need to mount a dedicated `/proc` for isolated process/PID awareness
  * `CLONE_NEWNET`: we will need to set up network interfaces dedicated to the namespace
  * Assigning the forked process ID to `/sys/fs/cgroup/pids/[users-process-id]/cgroup.procs`
* Our library will call this fork/wrapper binary with the pass-thru of the user command via `os/exec.Command` to fork and wire the streamed output via our `ProcessStore` implementation `Stdout` and `Stderr` `io.Writer` implementations to `os.exec.Command.Stdout` and `os.exec.Command.Stderr`
* This call to the fork will happen async, in a go routine, to ensure we can run it in the background, using a channel to communicate as necessary to the parent process in the library/the API process calling the library.

### How Stopping an Existing Process Works

The input to `Stop` is a service-specific worker ID. The library will take the following path to stop the related process if it can:

* The `ProcessStore` and result of `Start` will be a service-specific worker ID
* The `Stop` command will accept this service-specific worker ID, validate that the `requesterID` is allowed to stop the process, in our case only if its exactly the owner, retrieve the related host PID via the `ProcessStore.GetHostPID` implementation
* Then an `exec.Command` to `kill` will be launched for that PID
* Additionally we will perform cleanup for cgroup paths, `ProcessStore` entry, etc. and all other things that might not be captured by normal Golang garbage collection. This will be part of a resuable method that is utilized on completion/termination of a worker otherwise as well. 

## API

The API will be a gRPC-based API like the following proto definition for the service and request messages:

```
service Worker {
  rpc Start (StartRequest) returns (StartResponse) {}
  rpc Stop (StopRequest) returns (StopResponse) {}
  rpc GetStatus (GetStatusRequest) returns (GetStatusResponse) {}
  rpc StreamOutput(StreamOutputRequest) returns (stream StreamOutputResponse) {}
}

message StartRequest {
  string cmd = 1;
}

message StopRequest {
  string worker_id = 1;
}

message GetStatusRequest {
  string worker_id = 1;
}

message StreamOutputRequest {
  string worker_id = 1;
}
```

### Authentication via mTLS

Our CA, keys, and certs will use:

* TLS version 1.3
* Elliptical curve cryptography: prime256v1, ECDSA w/ sha256 signature algorithm

The above choices are aimed at balancing a combination of: simplicity, compatibility, security best-practices, and performance for this POC/MVP.

The CA/server cert and key will be generated as a one-time operation for use by the API. The tooling to do so will at least be minimally re-usable to support future rotation and re-generation of server and user certs alike.

Individual user/client certs/keys will be generated per user, with `subject organization = username`. These certs would be handed out to users. This will be the basis for not only authentication, but also **authorization**:

* The client will authenticate with a user-specific cert, and thus pass awareness of the cert's username to the server for assigning ownership of a worker
* The server can then restrict the given user to perform operations (stop/get status) against the workers they own only as a minimal authorization scheme

## CLI

Intended usage of the CLI:

```shell
linux-worker start \
  --host=[address of the API server] \
  --cert-key-path=[path to user certificate key to authenticate to the API] \
  --cert-path=[path to user certificate to authenticate to the API] \
  -- [arbitrary command with args to run]
```

For example:

```shell
$ linux-worker start --host=127.0.0.1:50001 --cert-key-path=./user-cert.key -cert-path=./user-cert.pem -- tail -f /var/log/syslog
{"worker_id": "91d1a3cf-9c56-4d30-bd45-b378c4080a8b"}
```

> :pencil: The ID may end up being something other than a UUID as stubbed above, e.g. constructed ID from underlying ps ID combined w/ the username/cert, or something similar

would communicate with the GRPC API server running at `127.0.0.1` to begin execution of the job `tail -f /var/log/syslog`. The output is a JSON representation of the job/worker that has started. Alternatively, if the request failed for some reason, we might see an error returned:

```shell
$ linux-worker start --host=inactivehost --cert-key-path=./user-cert.key --cert-path=./user-cert.pem -- tail -f /var/log/syslog
{"error": "Failed to connect to API at 'inactivehost'"}
```

```shell
linux-worker stop \
  --host=[address of the API server] \
  --cert-key-path=[path to user certificate key to authenticate to the API] \
  --cert-path=[path to user certificate to authenticate to the API] \
  --worker-id=[worker ID returned from the start subcommand]
```

```shell
linux-worker get-status \
  --host=[address of the API server] \
  --cert-key-path=[path to user certificate key to authenticate to the API] \
  --cert-path=[path to user certificate to authenticate to the API] \
  --worker-id=[worker ID returned from the start subcommand]
```

Example of the `get-status` command and result:

```shell
$ linux-worker get-status --host=127.0.0.1:50001 --cert-key-path=./user-cert.key --cert-path=./user-cert.pem --worker-id=91d1a3cf-9c56-4d30-bd45-b378c4080a8b
{"exit_code": "", "status": "running"}
```

Other possible `get-status` examples:

```json
{"exit_code": "1", "status": "errored"}
```

```json
{"exit_code": "0", "status": "done"}
```

```shell
linux-worker stream-output \
  --host=[address of the API server] \
  --cert-key-path=[path to user certificate key to authenticate to the API] \
  --cert-path=[path to user certificate to authenticate to the API] \
  --worker-id=[worker ID returned from the start subcommand]
```

Example of a streaming `stream-output` command and result to completion:

```shell
$ linux-worker stream-output --host=127.0.0.1:50001 --cert-key-path=./user-cert.key --cert-path=./user-cert.pem --worker-id=91d1a3cf-9c56-4d30-bd45-b378c4080a8b
{"stdout": "log line 1", "stderr": ""}
{"stdout": "log line 2", "stderr": ""}
{"stdout": "", "stderr": "something went wrong, retrying..."}
{"stdout": "complete", "stderr": ""}
$
```

* The output will be streamed in JSON-log format, the cli command stream only ending with a common termination signal, or when the worker is done and the stream of output is complete

## How Streaming Output Requirements Are Met

The requirements for streaming output of a running job, or retrieving full logs of a completed job:

* Output should be from start of process/worker execution.
* Multiple concurrent clients should be supported.

We will look at how this is supported from the service as a whole and how the pieces fit together:

* An implemented `ProcessStore` will assign an `io.Writer` implementation to the `exec.Command` `Stdout` and `Stderr`. This means that the internals of `os/exec` will make sure that the stdout/stderr of the executed command will write to the `ProcessStore` `Stdout` and `Stderr` objects for a given process.
* In our case, we're implementing an MVP/POC memory `ProcessStore` that uses `bytes.Buffer` as these `io.Writer` implementations. And these `bytes.Buffer` objects are pointers so that they can be pointers to actual places in memory for writing/reading directly. And the byte buffers should store both stdout and stderr from the very beginning of the user process output.
* Our memory `ProcessStore` implementation will do the necessary conversion of the `io.Writer` to `io.Reader` to ensure the library `GetOuput` connects the two via `io.Pipe` and converting to a custom `io.Reader` implementation that support mutex-based concurrent reads from the `io.Reader` implementation.
* For our `rockholla-go-linux-ns-cgroup-worker-fork`, we will connect its `Cmd.Stdout` and `cmd.Stderr` to the `ProcessStore` `io.Writer` implementations to make sure that connection is made to the isolated namespace/cgroup process output
* The API can then use the `io.Reader.Read` from the library `GetOutput` to convert that as necessary to a gRPC stream