SHIELD v9
=========

This is a **design document** intended to help flesh out the
properties of the SHIELD v9 data protection system, before
engineering work begins.

Components
==========

A SHIELD installation consists of 3 types of systems:
_Schedulers_, _Storage Gateways_, and _Agents_.

The job of the _Scheduler_ is to maintain the configuration of
protected systems, determine when backup jobs are to be run,
execute and orchestrate all backup / restore operations, and track
snapshots and their retention.

A _Storage Gateway_ sits in front of one or more storage backends
(i.e., Amazon S3, WebDAV, BackBlaze, etc.) and offers secure
access to blobs.  It is solely responsible for the compression,
encryption, storage, and retrieval of data snapshots.

An _Agent_ handles the communication with one or more protected
systems, to extract the data from those systems into point-in-time
snapshots.

Communication Flows
===================

The operation of SHIELD occurs along the following four
commications paths:

  1. Scheduler → Storage Gateway
  2. Scheduler → Agent
  3. Agent → Scheduler
  4. Agent → Storage Gateway

### Scheduler → Storage Gateway

The _Scheduler_ talks to the _Storage Gateway_ for two reasons:
storage enumeration and store/retrieval provisioning.

### Scheduler → Agent

### Agent → Scheduler

### Agent → Storage Gateway


Static Configuration
====================

### Storage Gateways

_Storage Gateways_ are configured statically with the list of
backing storage systems, and the credentials for accessing them.

    ---
    # /etc/shield/gateway.yml
    shield: v9
    storage-gateway:
      cluster: main
      bind: *:9999

      control:
        username: admin
        password: $ADMIN_PASSWORD

      defaultBucket:
        maxSnapshotRetention: 14d
        compression: lzma
        encryption: aes256-gcm

        vault:
          kind: hashicorp
          hashicorp:
            url: https://vault.corp/
            prefix: /secret/foo/shield/stuff   # default to /secret/shield
            roleID: foo
            secretID: bar...
            caCertificate:
              literal: $VAULT_CA

      buckets:
        - key: s3-cheap               # <-- for SSG API control interaction
          name: Nearline (Compressed) # <-- for Job dropdown store selection
          description: |
            Cheap, fast, always-available cloud storage,
            on Amazon AWS, with ZERO dependencies on our
            VPCs, cross-connects, and backhaul network.

            These snapshots are encrypted at rest by
            Jeff Bezos himself.

          maxSnapshotRetention: 90d
          compression: xz
          encryption: none

          # ^^ public information

          config: &s3-cheap
            vault: &kms # this overrides COMPLETELY
              kind: amazon
              amazon: ...

           kind: s3
            s3:
              region: us-east-1
              bucket: all-our-shield-backups
              aki: AKI....
              key: sekrit....
              prefix: /backups

        - key: s3-cheap-uncompressed   # <-- for SSG API control interaction
          name: Nearline (!Compressed) # <-- for Job dropdown store selection
          description: |
            Cheap, fast, always-available cloud storage,
            on Amazon AWS, with ZERO dependencies on our
            VPCs, cross-connects, and backhaul network.

            (This is an uncompressed storage)

          config: *s3-cheap # bring in all the details from the other store

          maxSnapshotRetention: 90d
          compression: xz
          encryption: aes256-ctr

        - key: s3-cheap-secure
          name: Nearline (Secure)
          description: |
            Cheap, fast, always-available cloud storage,
            on Amazon AWS, with ZERO dependencies on our
            VPCs, cross-connects, and backhaul network.

          retired: yes # active for downloads / expunge only
                       # and it won't show up in /storage

          maxSnapshotRetention: 90d
          compression: none
          encryption: aes256-gcm

          vault: *kms

          kind: s3
          s3: *cheap

        - key: enclave1
          name: Enclave
          description: |
            Secure Data Enclave, inside the confines of
            our Atlanta data center(s).  Please use this
            for data under compliance (PCI, HIPAA, etc.)

          kind: webdav
          webdav:
            url: https://secure.data.enclave:9097/nas13
            username: $NAS_USERNAME
            password: $NAS_PASSWORD
            caCertificate:
              file: /path/to/file

### Scheduler

    ---
    # /etc/shield/scheduler.yml
    shield: v9
    scheduler:
      identity: My Awesome SHIELD
      bind:     127.0.0.1:8080

      storage:
        - cluster: main
          conflict: majorityWins
          endpoints:                     # we can explicitly
            - http://ssg1:9997           # list out our SSG
            - http://ssg2:9998           # endpoints...
            - http://ssg3:9999
          fromDNS:                       # or look them up
            lookup: ssg1.example.com     # (periodically)
            template: https://{IP}:9999  # via DNS

      database:
        kind: postgres
        postgres:
          host: $PG_HOST
          port: $PG_PORT
          username: $PG_USER
          password: $PG_PASS
          database: shield1
          options:
            - sslmode=nothanks

      redis:
        host: $REDIS_HOST   # 127.0.0.1
        port: $REDIS_PORT   # 6379
        auth: $REDIS_AUTH   # ""

      vault:
        kind: hashicorp
        hashicorp:
          url: https://vault.corp/
          root: /secret/shield/creds
          roleID: foo
          secretID: bar...
          caCertificate:
            literal: $VAULT_CA

### Agent

    ---
    # /etc/shield/agent.yml
    shield: v9
    agent:
      identity: agent@some.host      # default: "shield@$HOSTNAME"

      privateKey:
        file: /path/to/private.key   # default: "/etc/shield/agent.key"

      scheduler:
        host: scheduler.shield.tld   # default: ${SHIELD_HOST}
        port: 2222                   # default: ${SHIELD_PORT:-2222}

        hostKey:                     # explicitly trust known
          file: /path/to/host.pub    # host public keys...

          provision:
            url: https://shield      # ... or provision one from
            caCertificate:           # the SHIELD scheduler...
              literal: $SHIELD_CA

          trustAll: true             # .. or trust anyone

      variables:                     # there are no defaults for
        - name: PASSWORD             # defined variables / secrets
          value: foobar
          description: |
            The PostgreSQL password, for accessing the database
            over loopback (UNIX domain socket!) only.

      exec:                          # nor are there any defaults
        - name: pgbackup             # for exec-based "plugins"
          env:
            PGDATABASE: $PGDB
            PGPORT: 9011
          backup:
            command: pgdump -C -c
          restore:
            command: psql


Database Schema
===============

This section looks at the database schema, in non-specific terms,
to better understand what live configuration SHIELD supports.


### Users

| Field          | Type   | Notes |
| -------------- | ------ | ----- |
| `id`           | ID     | A unique, fixed identifier for this user.  Does not change. |
| `username`     | string | A unique username for this account. |
| `display_name` | string | A display name for this account. |
| `pwhash`       | string | An opaque password hash for validating password equivalence. |
| `created_at`   | time   | When the user account was created. |
| `last_seen_at` | time   | When the user account last authenticated. |
| `grants`       | object | A description of granted access within this SHIELD. |


### Agents

| Field             | Type    | Notes |
| ----------------- | ------- | ----- |
| `id`              | ID      | A unique, fixed identifier for this agent.  Does not change. |
| `identity`        | string  | Identity supplied by the agent. |
| `fingerprint`     | string  | A fingerprint of the private key, for validation purposes. |
| `notes`           | string  | An optional annotation for this snapshot, interpreted as Markdown. |
| `authorized`      | boolean | Is this agent authorized to operate in the SHIELD? |
| `authorized_at`   | time    | When this agent was authorized (if it ever was). |
| `deauthorized_at` | time    | When this agent was deauthorized (if it ever was). |
| `last_seen_at`    | time    | When the SHIELD scheduler last communicated with the agent. |
| `execs`           | object  | A map of pre-configured exec "plugins" for snapshot / replay. |
| `variables`       | list    | A list of defined variables (without their values) that the agent knows about, locally. |
| `version`         | string  | The version of the SHIELD agent software. |
| `capabilities`    | object  | A description of the capabilities of this agent. |
| `admin_status`    | enum    | The administrative status of this agent. |
| `health_status`   | enum    | The health status of this agent. |
| `ping_interval`   | number  | How often, in seconds, to ping this agent for liveness. |
| `ping_tolerance`  | number  | How many pings can this agent miss before it is considered offline.  Should default to 0. |

Agent administrative status must be one of the following:

  - `unauthorized` - The agent has not been granted access to operate in this SHIELD.
  - `authorized` - The agent is allowed to receive task assignments.
  - `suspended` - The agents authorization has been temporarily suspended, and it should not be given task assignments.

Agent health status must be one of the following:

  - `pending` - The agent has never connected to this SHIELD.
  - `online` - The agent is connected to the underlying SSH fabric.
  - `unresponsive` - The agent has missed at least one heartbeat ping, but has not missed enough to be considered offline.
  - `offline` - The agent either disconnected gracefully, or has missed enough heartbeat pings to be considered offline and forcibly disconnected.


### Systems

| Field         | Type   | Notes |
| ------------- | ------ | ----- |
| `id`          | ID     | A unique, fixed identifier for this system.  Does not change. |
| `agent_id`    | ID     | The ID of the agent that handles this system. |
| `name`        | string | A human-friendly name for this system.  Subject to change. |
| `description` | string | A longer description of this system, interpreted as Markdown. |
| `kind`        | string | What logic should be used to take / replay snapshots? |
| `created_at`  | time   | When this system was originally created. |
| `updated_at`  | time   | When this system was last modified. |
| `retired_at`  | time   | When this system was taken out of service. |
| `deleted_at`  | time   | When this system was deleted.  Deleted objects stay in the database, but do not show up to most queries. |
| `parameters`  | object | A single-level key-value object of all configured parameters.  Depends entirely on the value of `kind` for semantics. |
| `folder`      | string | Absolute path to the "folder" into which this system should be displayed, for hierarchical organization schemes. |


### Snapshots

| Field             | Type    | Notes |
| ----------------- | ------- | ----- |
| `id`              | ID      | A unique, fixed identifier for this snapshot.  Does not change. |
| `system_id`       | ID      | The ID of the system that this snapshot belongs to. |
| `schedule_id`     | ID      | The ID of the schedule that originated this snapshot. |
| `notes`           | string  | An optional annotation for this snapshot, interpreted as Markdown. |
| `starred`         | boolean | Whether or not an operator has starred this snapshot. |
| `created_at`      | time    | When this snapshot was taken. |
| `expires_at`      | time    | When this snapshot will expire, and can be removed from sturage. |
| `purged_at`       | time    | When this snapshot was actually purged from storage. |
| `deleted_at`      | time    | When this snapshot was deleted.  Deleted objects stay in the database, but do not show up to most queries. |
| `compression`     | string  | A code indicating the compression algorithm used. |
| `encryption`      | string  | A code indicating the encryption algorithm used. |
| `raw_size`        | number  | Size (in bytes) of the uncompressed data. |
| `compressed_size` | number  | Size (in bytes) of the compressed data. |
| `location`        | string  | A (custom) URL indicating where snapshot data is stored. |
| `status`          | enum    | The status of this snapshot. |

Snapshot URLs take the form:

    shield://$gateway/$bucket/$key

where:

  - `$gateway` is the name of a SHIELD Storage Gateway cluster
  - `$bucket` is the name of a storage bucket configuration in the named cluster
  - `$key` is a backend-specific identifier for the snapshot data.  In S3, this is a path-like key that may contain slashes, i.e. `2020/05/24/backup1.snp`

Snapshot status must be one of the following:

  - `allocated` - This snapshot has been allocated in the storage gateway, but has not been finalized (i.e. the upload step hasn't completed)
  - `valid` - The snapshot is in storage, and is valid and not yet expired.
  - `expired` - The snapshot is still in storage, but has passed its expiration date, and should be purged.
  - `purging` - The snapshot is being purged from the storage gateway.
  - `purged` - The snapshot is no longer resident in the storage gateway, and cannot be replayed.
  - `HOLD` - An operator has placed an administrative hold on the snapshot, to prevent it from being purged, even after its expiration.


### Schedules

| Field              | Type    | Notes |
| ------------------ | ------- | ----- |
| `id`               | ID      | A unique, fixed identifier for this schedule.  Does not change. |
| `system_id`        | ID      | ID of the owning System. |
| `original_spec`    | string  | The original time specification, as given by the operator. |
| `parsed_spec`      | object  | The parsed time specification, in object (JSON) form. |
| `created_at`       | time    | When this schedule was originally created. |
| `updated_at`       | time    | When this schedule was last modified. |
| `last_run_at`      | time    | When the most recent task scheduled. |
| `next_run_at`      | time    | When the next task should be scheduled. |
| `retired_at`       | time    | When this schedule was taken out of service. |
| `deleted_at`       | time    | When this schedule was deleted.  Deleted objects stay in the database, but do not show up to most queries. |
| `retain_min_count` | number  | The minimum number of snapshots to retain. |
| `retain_max_count` | number  | The maximum number of snapshots to retain. |
| `retain_min_days`  | number  | The minimum number of days to retain a snapshot. |
| `retain_max_days`  | number  | The maximum number of days to retain a snapshot. |
| `status`           | enum    | The status of this schedule. |
| `store`            | string  | The URL of the storage backend to store snapshots in. |

Schedule status must be one of the following:

  - `active` - The schedule is in force, and tasks will be scheduled according to its time specification.
  - `paused` - The schedule is temporarily not in force, and no task scheduling will be done according to its parameters.
  - `retired` - The schedule is permanently not in force, and no task scheduling will be done according to its parameters.
  - `deleted` - The schedule has been deleted.

Storage URLs take the form:

    shield://$gateway/$bucket

where:

  - `$gateway` is the name of a SHIELD Storage Gateway cluster
  - `$bucket` is the name of a storage bucket configuration in the named cluster



### Tasks

| Field              | Type    | Notes |
| ------------------ | ------- | ----- |
| `id`               | ID      | A unique, fixed identifier for this task.  Does not change. |
| `operation`        | enum    | The type of task. |
| `status`           | enum    | The status of this task. |
| `system_id`        | ID      | The ID of the system that this task pertains to. |
| `schedule_id`      | ID      | The ID of the schedule that originated this task (only applicable to `snapshot` operations). |
| `snapshot_id`      | ID      | The ID of the snapshot. |
| `notes`            | string  | An optional annotation for this task, interpreted as Markdown. |
| `created_at`       | time    | When the task was created (requested). |
| `cancelled_at`     | time    | When the task was canceled by an operator (if it ever was). |
| `started_at`       | time    | When the task was successfully dispatched to its agent. |
| `stopped_at`       | time    | When the task stopped executing (either due to failure, or successful completion). |

Task operation must be one of the following:

  - `snapshot` - A snapshot of the indicated system is to be performed to the given store.  A snapshot will be pre-allocated to hold the data, so that we (the scheduler) can purge it should the agent crash halfway through a snapshot.
  - `replay` - The indicated snapshot should be retrieved from storage and replayed to the given system.

Not all Snapshot fields have meaning; some of them only make sense in the context of a specific type of task (as denoted by `operation`).  Namely:

| Operation    | system_id | schedule_id | snapshot_id |
| ------------ | --------- | ----------- | ----------- |
| **snapshot** |    yes    |      yes    | (allocated) |
| **replay**   |    yes    |     _no_    |    yes      |

Task status must be one of the following:

  - `pending` - The task has been submitted, but has yet to be turned into an executing process by the scheduling core.  Tasks may stay in pending for a long time if, for example, the agent responsible for them is not found or because of concurrency concerns.  SHIELD will not, without explicit logic dictating otherwise, allow two or more tasks to execute against a single data system.
  - `running` - The task has been assigned and dispatched to its agent, and is currently executing.
  - `errored` - An internal error occurred, either within the agent software, or the scheduler software, that prevented the task from finishing successfully.
  - `failed` - An external error occurred, either a loss of connectivity to the storage gateway, unability to communicate with the targeted data system, etc.
  - `completed` - The task ran to completion without error.
  - `canceled` - The task was administratively canceled by an operator.
  - `handled` - A non-OK status has been manually acknowledged by an operator, and should not be considered an error (for the sake of monitoring / alerting / display.)

The `created_at` timestamp is guaranteed to exist, and can never be nil.

The `canceled_at` timestamp represents when cancellation was _requested_, not when it was _realized_ -- the latter timestamp is stored in `stopped_at`.

| Status    | started_at | stopped_at | Notes |
| --------- | ---------- | ---------- | ----- |
| pending   | nil        | nil        | Task has not been dispatched to its agent yet. |
| running   | not nil    | nil        | Task has been dispatched and is executing. |
| errored   | nil        | not nil    | An internal error occurred in the scheduler before we could dispatch to an agent for execution. |
| errored   | not nil    | not nil    | An internal error occurred in the agent, after we dispatched to it for execution. |
| failed    | not nil    | not nil    | An external error occurred. |
| completed | not nil    | not nil    | Task has completed successfully. |
| canceled  | *          | nil        | Task cancellation has been requested, but not fully realized. |
| canceled  | nil        | not nil    | Task was canceled before it could be dispatched to an agent. |
| canceled  | not nil    | not nil    | Task was canceled after it was dispatched to an agent. |

No other combinations of (status, (nil? started\_at), (nil? stopped\_at)) are legal.


SHIELD APIs
===========

The SHIELD Scheduler and Storage Gateway components each have their own distinct but uniform APIs.  The Scheduler API paths are all prefixed with `/v9/scheduler`, and the Storage Gateway API paths all start with `/v9/storage-gateway` -- this unconventional nomenclature allows us to colocate the two components in a single endpoint, should operators desire that.

The Scheduler API is fully defined in the `api/scheduler.yml` Swagger definition that accompanies this README file.  Likewise, the Storage Gateway API is defined in `api/storage-gateway.yml`.
