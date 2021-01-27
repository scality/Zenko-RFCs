# Ballot Launcher

## Overview

This specification presents the design for `ballot`: a coordinated
task-launching utility.

## Problem

The S3C stack currently lacks cluster-wide coordination of maintenance tasks and
singleton services. This creates issues such as:

- Single points of failure - if the server singled out to run a task (either
  randomly or because it's the first of a list) becomes unavailable, then the
  task is not run as expected.
- Risk of concurrent execution of tasks that may interfere with each other.
- Processing load may be uneven.
- Knowing what will run where (and what has run where) requires looking at logs
  and configuration.

## Use Cases

### Tasks

The following task types will benefit from coordination:

- Scheduled tasks (cron jobs), either on a timetable or an interval between
  expected runs.
- Always-on services or long-running one-shot commands that should run only one
  copy for data consistency or resource allocation reasons.

### Use Case for Developers

As a developer integrating a new processing job that fits the Impacted Tasks
definition above, I need a way for the deployment tool (Federation) to schedule
running the script on all eligible servers without any special treatment at
install time, with a guarantee that it will not run on multiple servers at the
same time, but will still run no matter which servers are down.

### Use Case for Operations

As an operations person, I need to be sure that maintenance jobs:

- Run when expected regardless of system state.
- Can be easily inspected.
- Can be paused if system state requires it.

## Technical Details

### Principle

A launcher, `ballot` is introduced and used to run commands. Before executing
the provided command, `ballot` tries to aquire leadership on a ZooKeeper path
specific to the command (using the standard leader election recipe). The leader
then runs the command, while followers wait. When the leader resigns, the next
follower in line takes over leadership and runs the command.

### Proposed UX

To give a clear idea of expected capabilities, the envisioned user experience is
described below.

#### Run a Simple Task

```shell
ballot --zookeeper-path /com/scality/tasks/mycluster/consolidate \
    run once --candidate-id dc01-room01-rack01-server01 -- python consolidate.py
```

#### Run a Scheduled Task

```shell
ballot --zookeeper-path /com/scality/tasks/mycluster/consolidate \
    run cron --schedule @hourly --candidate-id dc01-room01-rack01-server01 -- python consolidate.py
```

#### Show Information on a Task

Display information on a distributed task. Useful information that may be
displayed includes: current leader's candidate ID, list of follower's candidate
IDs, date and time of election, hostname, PID.

```shell
ballot --zookeeper-path /com/scality/tasks/mycluster/consolidate info
```

#### Pause Execution of a Task

To conserve resources during an outage or planned maintenance (just disabling
future elections, not impacting currently running jobs):

```shell
ballot --zookeeper-path /com/scality/tasks/mycluster/consolidate pause
```

To terminate currently running jobs:

```shell
ballot --zookeeper-path /com/scality/tasks/mycluster/consolidate pause --terminate
```

#### Resume Execution of a Task

To resume a paused job:

```shell
ballot --zookeeper-path /com/scality/tasks/mycluster/consolidate resume
```

#### List All Known Tasks

To show a cluster-wide list of tasks and their status (info such as which node
is the leader, a list of followers, election time, hostname, PID).

```shell
ballot --zookeeper-path /com/scality/tasks/mycluster status
```

#### Integration points

`ballot` can be used as a direct `ENTRYPOINT` in Docker images, as a
command-line prefix for containers, or as a trampoline in supervisord
configuration files, whichever is least intrusive.

### Changes Introduced

Where tasks were assigned to one server at deployment time, they instead are
configured to run on all eligible servers, and use `ballot` as a launcher, which
ensures uniqueness and fallback.

### Interaction with Other Components

Ballot requires a functioning ZooKeeper ensemble to operate.

Given its limited responsibilities, other components/tools are required for
general operation, such as:

- `/usr/bin/env` for setting environment variables (e.g. `ballot run once -- env
  VAR=value python script.py`).
- Log shipping/aggregation as configured in S3C - `ballot` will pass through the
  task's logs.
- An always-up process manager (Docker, supervisord) to handle log redirection
  to a known destination, restart and backoff decisions for the `ballot` process
  tree.

### Monitoring

Monitoring a command run with `ballot` should be the same as monitoring the same
command run without ballot; no additional burden should be added. To achieve
this, the commands's logs and exit code can be passed through `ballot`.
Launcher-specific issues can be interpreted as a task failure.

Monitoring of the task itself should continue as usual, and monitoring of
ZooKeeper becomes more crucial as more workloads depend on it being available.

For later consideration: `ballot` could include some options to integrate with
existing alerting mechanisms.

### Failure Scenarios

#### Leader Election Failure

If leader election fails for any reason (ZooKeeper down or unreachable, bad
configuration, ...), the following resolutions can be chosen from:

- `retry`: Retries the election process until a leader is successfully elected.
- `exit`: Exits the `ballot` process and lets the underlying process manager
  (`docker`, `supervisord`, ...) apply its restart and backoff policy.
- `run-anyway`: Ignores the failure and runs the command anyway. This is useful
  in case it's more important to have the command run, even if it is run
  multiple times in parallel, than for it to not run at all.

#### Task Failure

Tasks can fail for reasons such as unhealthy host, network issue/partition,
application bug w.r.t. bad data handling, etc. No unique solution would work, so
a range of resolutions can be chosen from:

- `reelect`: Drops leadership so that a new leader election can take place,
  potentially allowing a more healthy host to take over.
- `rerun`: Runs the command again immediately, and repeats until the command
  succeeds.
- `exit`: Exits the `ballot` process and lets the underlying process manager
  Docker, supervisord, ...) apply its restart and backoff policy.
- `ignore`: Ignores the failure and exits with a success code anyway. In cron
  mode, ignores the failure and waits for the next scheduled trigger.

Note: failures to execute the command (like a missing binary) are handled by the
same mechanism as failure of a command that was `exec`'d successfully. In this
case, a failure policy of `reelect` would make sense while `rerun` would not be
of much help.

Note: for flexibility, the same actions can be performed on successful
completion of the child process.

#### ZooKeeper Failure after Leader Election

If ZooKeeper loses quorum, crashes, or becomes unreachable, `ballot` will try to
reconnect in the background, and in the meantime not take any action:

- The leader still considers itself the leader, and carries on with waiting for
  its child process completion. In cron mode, it continues scheduling.
- Any followers will continuously try to become leaders (or be restarted)
  according to the Leader Election Failure policy described above, and should
  not succeed.

Upon reconnection to ZooKeeper, `ballot` ZooKeeper session IDs are matched to
the owners of the ephemeral proposal znodes, picking up the leader/follower
roles as they were before the disruption. If any `ballot` process was started
after the disruption, then it will automatically join as a follower, as long as
some `ballot` processes with longer-running session IDs are present.

In the special case where ballot processes are configured to `exit` or `reelect`
and are not run in `cron` mode, once all commands are done running, nothing will
run until ZooKeeper is back online, which can be seen as the worst-case
scenario.

## Alternatives

### Other Technologies

Tools such as [`lockrun`](http://www.unixwiz.net/tools/lockrun.html) are often
used by sysadmins and provide the solution to the "at most once" problem on a
single server/container, but still require coordination so that no single server
is considered special and can still be taken out of service without impacting
task availability.

Kubernetes provides resources that solve these problems such as `jobs`,
`cronjobs`, `statefulsets` with one replica. However S3C does not run on
Kubernetes.

[Dkron](https://dkron.io/) comes with its own Raft quorum, which adds deployment
and operation burden. Its free/commercial split license can be inconvenient.

[Chronos](https://github.com/mesos/chronos) is very complete and comes with good
reporting capabilities. It runs on top of Mesos, which makes it heavy and
impractical in the context of S3C.

### Other Approaches

#### Leader Election Functionality

Since the primary goal was to be as unintrusive and scalable (developer-time
wise) as possible, implementing a leader election in each task type was not a
viable option.

#### Cron Functionality

To keep things simple and focused, one approach considered was to not include
the `cron` functionality in `ballot`, and instead either:

- Have each targeted task implement the cron functionality itself. Ruled out
  because of code reusability concerns, and lots of boilerplate involved in the
  leader election.
- Run `ballot run my-command` in a crontab line. This is prone to overflowing
  ZooKeeper if executions were to stack up waiting for their turn. Implementing
  invocation tracking would also be required to make sure that once a scheduled
  invocation finishes successfully, no other should run for the same trigger.
- Run `ballot run anacron -t my-crontab-file`. This is close to a workable
  solution, but increases the work needed for integration as it does not allow
  "smart" retry policies, and pass-through of exit codes and logs.
