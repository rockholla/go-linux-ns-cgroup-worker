# Linux Worker Service using Namespaces/CGroups Implemented in Golang

This is an experiement to build a job worker service that provides an API to run arbitrary Linux processes.

There will be 3 components to make up this service:

1. A Golang library to start, stop, and query the status/output of a worker job
2. A gRPC API to expose the functionality of this library over a network to start, stop, query status and stream output of a job, including authentication via mTLS and authorization scheme
3. A CLI for users to interact with the API to start, stop, query status/stream output of a job

> :pencil: Note that code samples here are meant to be approximations of the final implementation. Slight variations may exist in what is actually implemented in code from what you see in this design.

## Library

This will be the lowest-level part of the service stack, with the following goals and outcomes:

* providing an interface for a linux command/process to be executed on behalf of a user/owner
* interfacing with the linux OS internals/kernel via [`sys/unix`](https://pkg.go.dev/golang.org/x/sys/unix) to:
  * Add process isolation for a worker process in the following ways:
    * **PID**: no other processes executed by the worker service can see another worker process
    * **mounts**: dedicate file system space for the process
    * **networking**: isolate network interface and connectivity out of the linux machine, but with no routes to other processes/namespaces
  * Add resource usage control by assigning the spawned process to a cgroup, and limiting the amount of:
    * **CPU used by the process**: hard limit on the total CPU that any single process could use from the host machine
    * **Memory used by the process**: hard limit on the total memory that any single process could use from the host machine
    * **Disk I/O rate used by the process**: hard limit on the read/write bytes/second available to any single process
* Returning a unique identifier for a successfully-triggered process/worker
* Ability to stop a running process by using this unique identifier
* Ability to get/stream the output/logs of the worker process by using the unique identifier, as well as get the final result exit code when available

### Library-Defined Process Store Interface

The library will define an interface for storing and tracking running processes, a process store, like:

```golang
// ProcessStore defines an interface that could define a way for the library to allow itself or others
// to define custom stores for tracking a processes output and status
type ProcessStore interface {
	NewProcess(processID string, ownerID string) (err error)
	GetOutputWriters(processID string) (stdout io.Writer, stderr io.Writer, err error)
  GetOwnerID(processID string) (ownerID string, err error)
	GetStatus(processID string) (stdout io.Reader, stderr io.Reader, done bool, exitCode int, err error)
}
```

Our first implementation of this interface will be part of the libary and as simple as possible: a memory store tracking the owner, stdout, stderr, whether complete, and exit code for any requested worker.

Our library method contracts are designed as follows:

```golang
// Start will initiate a new process/command on behalf of a user/owner
func Start(cmd []string, ownerID string, store ProcessStore) (processID string, err error)

// Stop will take in a process ID and the ID of the user requesting the stop, determine if a stop is allowed, and then do it
func Stop(processID string, requesterID string) error

// GetStatus is a wrapper around the process store implementation to also manage authorization-related logic, e.g. does requester = owner
func GetStatus(processID string, requesterID string) (stdout io.Reader, stderr io.Reader, done bool, exitCode int, err error)
```

## API

The API will be a gRPC-based API like the following proto definition:

```
service Worker {
  rpc Start(StartRequest) returns (StartResponse) {}
  rpc Stop(StopRequest) returns (StopResponse) {}
  rpc GetStatus(GetStatusRequest) returns (stream GetStatusResponse)
}
```

### Authentication via mTLS

Our CA, keys, and certs will use:

* TLS version 1.3
* Elliptical curve cryptography: prime256v1, ECDSA w/ sha256 signature algorithm

The above choices are aimed at balancing a combination of: simplicity, compatibility, security best-practices, and performance for this POC/MVP.

The CA/server cert and key will be generated as a one-time operation for use by the API. The tooling to do so will at least be minimally re-usable to support future rotation and re-generation of server and user certs alike.

Individual user/client certs/keys will be generated per user, with `subject organization = username`. This will be the basis for not only authentication, but also **authorization**:

* Server user store, just in-memory for now, will store known usernames for data association, e.g. processes owned by cert username
* The client will authenticate with a user-specific cert, and thus pass awareness of the cert's username to the server
* The server can then restrict the given user to perform operations (stop/get status) against the processes they own only as a minimal authorization scheme

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
$ linux-worker start --host=127.0.0.1 --cert-key-path=./user-cert.key -cert-path=./user-cert.pem -- tail -f /var/log/syslog
{"id":"91d1a3cf-9c56-4d30-bd45-b378c4080a8b"}
```

> :pencil: The ID may end up being something other than a UUID as stubbed above, e.g. constructed ID from underlying ps ID combined w/ the username/cert, or something similar

would communicate with the GRPC API server running at `127.0.0.1` to begin execution of the job `tail -f /var/log/syslog`. The output is a JSON representation of the job/process that has started. Alternatively, if the request failed for some reason, we might see an error returned:

```shell
$ linux-worker start --host=inactivehost --cert-key-path=./user-cert.key --cert-path=./user-cert.pem -- tail -f /var/log/syslog
{"error":"Failed to connect to API at 'inactivehost'"}
```

```shell
linux-worker stop \
  --host=[address of the API server] \
  --cert-key-path=[path to user certificate key to authenticate to the API] \
  --cert-path=[path to user certificate to authenticate to the API] \
  --process-id=[process ID returned from the start subcommand]
```

```shell
linux-worker get-status \
  --host=[address of the API server] \
  --cert-key-path=[path to user certificate key to authenticate to the API] \
  --cert-path=[path to user certificate to authenticate to the API] \
  --process-id=[process ID returned from the start subcommand]
```

Example of a streaming `get-status` command and result to completion:

```shell
$ linux-worker get-status --host=inactivehost --cert-key-path=./user-cert.key --cert-path=./user-cert.pem --process-id=91d1a3cf-9c56-4d30-bd45-b378c4080a8b
{"stdout": "log line 1", "stderr": "", exit_code: ""}
{"stdout": "log line 2", "stderr": "", exit_code: ""}
{"stdout": "final log line", "stderr": "", exit_code: "1"}
$ 
```

* The output will be streamed in JSON-log format, the `get-status` cli command stream only ending with a common termination signal, or when the process is done and the stream of output is complete.
* The absence of a populated `exit_code` means the process is still running from the user's perspective
* Conversely, the user of the command output can parse the `exit_code` from the final line of the output from this command