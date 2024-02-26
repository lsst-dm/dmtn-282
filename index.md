# Butler Client/Server Design Meeting October 2023

```{abstract}
This tech note describes the initial design of the Butler client/server presented at a meeting held over three days at Princeton in October 2023.
This builds on ideas previously considered and covered in [DMTN-242](https://dmtn-242.lsst.io) and [DMTN-249](https://dmtn-249.lsst.io).
It also includes the initial implementation plan agreed at that meeting.
```

## Introduction

The Rubin Data Butler was formally delivered by the construction project in 2022 {cite:p}`DMTR-271`.
For operations the requirements were modified with the switch to a hybrid data access centre ([DMTN-240](https://dmtn-240.lsst.io); {cite:p}`DMTN-240`) where the data are served from SLAC but the users are logged into the Rubin Science Platform on Google.
To handle this change the Butler client/server concept was developed but it has required some major conceptual changes in how we handle user interfactions with a data repository.
These changes were considered in [DMTN-242](https://dmtn-242.lsst.io) {cite:p}`DMTN-242` and [DMTN-249](https://dmtn-249.lsst.io) {cite:p}`DMTN-249` and resulted in a decision to have a design meeting with the core developer team of Jim Bosch, Tim Jenness, Andy Salnikov, and Nate Lust, with additional input from Russ Allbery.
This meeting was held at Princeton University 2023 October 3 to 2023 October 5.
This document contains the discussion topics and vision that were used to seed the meeting, an initial plan for a minimal viable product (MVP) client/server deployment, and a Q&A session with Russ Allbery who provided his expertise on issues such as authentication and URL signing.

## Discussion Topics

### Fundamental goals/requirements

* Butler construction pattern does not change, even if the return value is a subclass instance.
* Butler repository construction pattern (`Butler.makeRepo`) does not change.

### Versioning and stability

* How thin can we make the server to increase its stability and reduce the frequency of deployments?
* Lots of caching needs to go to the client.
  Maybe not all, and server-side caching might need to be very different.

### Prototype Proposal

The following is a collection of thoughts written mostly by Jim Bosch for everyone to read before the meeting.

* A data repository has a database and a datastore (which may be chained, of course).
* New data repository consistency invariants:
    * A dataset may be registered in the database and be stored (i.e., have artifacts) in the datastore only if there are opaque records in the database related to that dataset OR if the dataset is managed by an artifact transaction.
    * A dataset may be registered in the database only (and not be stored) only if there are no opaque records in the database related to that dataset.
    * A dataset may be stored in the datastore without being registered in the database only if it is managed by an artifact transaction.
* An artifact transaction on a data repository is a limited-duration but potentially persistent (across many client sessions) that registers itself with the database and writes to the same datastore or a "related" datastore whose artifacts may be ingested by the repository datastore with `transfer="direct"`.
  An artifact transaction may be committed or abandoned, and it maintains repository consistency invariants even in the presence of hard errors like hardware problems.
* These invariants mean we only write artifacts by first opening an artifact transaction, then writing artifacts to datastore, then inserting opaque records into the database as we close the transaction.
* A workspace is an environment that can provide a butler-like interface sufficient for running pipelines.
    * An internal pipeline-execution workspace that writes directly to the repository’s datastore is associated with a long-lived artifact transaction that is opened when the workspace is created, and it is committed to "bring home" output datasets.
    * An external pipeline-execution workspace that writes elsewhere creates an (ideally) short-lived artifact transition only when it the workspace is committed, and it closes that transaction when the transfer back to the repository is complete.
    * Note that in [DMTN-271](https://dmtn-271.lsst.io) {cite:p}`DMTN-271`, Jim Bosch combined the "workspace" and "artifact transaction" concepts, which he thinks was a mistake, given how different internal and external workspaces are from the perspective of the central data repository.
* All active artifact transactions must be committed or abandoned before a schema migration or repository configuration change
* New component division of responsibilities:
    * `LimitedButler` is largely unchanged, and it’s still an ABC.
        * It’s what’s needed for `PipelineTask` execution, no more.
        * It may shrink a bit (no need to unstore) if clobbering-when-running is replaced by telling a workspace to reset quanta.
        * It may shrink a bit further if workspace quantum-status files replace is_stored checks.
    * `QuantumBackedButler(LimitedButler)` would be the implementation used by certain workspaces, which would eventually become the only legal way to execute tasks.
    * `Datastore` would remain an ABC but:
        * it would lose responsibility for storing its own opaque records; it instead returns them when writing and is given them prior to read (it would provide an implementation of the `OpaqueRecordSet` ABC for this, which would replace some or all of the opaque-table manager hierarchy we have now);
        * it would eventually be able to drop support for "prediction" mode, except for the ability to clean up artifacts after a failed sequence of writes that prevented records from being saved.
    * `Registry` would cease to be an ABC and cease to be a public interface; it would have one direct SQL implementation.
      It might even disappear completely (which is why we’ve talked about repository’s having databases, not registries), if we decide it’s more useful to just have a struct of `Manager` objects now that registry interfaces are purely internal (aside from the backwards-compatibility proxy, which isn’t really a registry).
    * `Butler(LimitedButler)` would become an ABC that represents any client for a full data repository (not just a workspace).
        * A `Butler` always has a `Datastore` (as package-private attribute: helper classes can assume this, but not external users).
        * `Butler.__new__` would be a factory for subclasses of `Butler`, but `LimitedButler` specializations like `QuantumBackedButler` that do not inherit from `Butler` would have their own construction patterns.
    * `DirectButler(Butler)` would be where our current Butler implementation more or less goes; it has a `Registry` as well as the `Datastore` and it inherits from `Butler`.
    * `RemoteButler(Butler)` is the client for the client/server butler.
        * Since it’s a `Butler`, `RemoteButler` has a `Datastore` on the client.
        * The server also has a `Datastore` constructed from the same configuration, but this one will be used only to verify that datastore artifacts written by the client-side `Datastore` and opaque records provided by the client-side `Datastore` for insert are consistent, which for files means existence checks and checksums.
        * The server has a `Registry`, or at least its constituent managers.
        * In our case, this `Datastore` will need to accept signed URIs to read and write, but these are provided by the `Butler` via its `OpaqueRecordSet` type.
          Whether this is new functionality on `FileDatastore`  or a new class that extends or otherwise delegates to `FileDatastore` is unspecified.
* `ArtifactTransaction` is a new helper ABC.
  Subclasses might represent put, transfers, deletes, or pipeline execution.
    * An open artifact transaction is represented in the database by a simple pair of tables that lock artifact writes on a per-run basis (only one transaction may write to a run at a time).
        * For prompt processing, we will likely want multiple processes to be able to gracefully race when opening an identical transaction, with all workers informed whether they were the first, so the first can then perform special one-time-only setup.
        * This will require some kind of atomic-write primitive on `ResourcePath` that guarantees that identical racing write attempts result in the file successfully being written.
            * S3 does this naturally as long as we don’t wreck it.
            * On POSIX we write a temporary file then mv.
            * What about WebDAV/HTTP?
            * We don’t need to be able to detect which writer won.
    * An open artifact transaction is also represented by one or more JSON files (though this could implemented in a NoSQL DB instead) that store transaction-type-specific content.
    * Construct an `ArtifactTransaction` with subclass-specific arguments (e.g., "these are the refs I want to delete").
    * Give an `ArtifactTransaction` to a `Butler` method to start a transaction and register it with a name; this serializes the `ArtifactTransaction` object to a JSON file (again, could have a NoSQL DB implementation, too) and inserts into the artifact transaction DB tables.
    * Give the name to other `Butler` methods to abandon or commit the transaction.
    * `DirectButler` accepts any `ArtifactTransaction`, trusting that object’s implementation to maintain repository invariants (subject to filesystem and database access controls).
    * `RemoteButler` accepts only certain predefined `ArtifactTransaction` objects that it recognizes: those for put, import/transfer/ingest, and deletion.
      It does not support the long-lived internal pipeline-execution transactions.
        * When the transaction is opened, the transaction object is serialized (one of the things an `ArtifactTransaction` knows how to do) and sent to the server.
          The server checks permissions and saves the transaction to a file and registers it with the DB.
        * On commit or abandon (these are mostly symmetric), the artifact transaction object on the client is given a datastore and the signed URIs it says it needs to do its work, returning the opaque records it wants to insert.
          When it is done, the server asks its version of the transaction object to verify what the client has done, and on success inserts the opaque records into the database and closes the transaction.
    * A batch of database-only operations can be run both when a transaction is opened and when it is committed or abandoned, in order to define mostly-atomic high-level operations like removing an entire run full of stored datasets.
    * In order to maintain our invariants, an artifact-deletion transaction actually deletes the opaque records when the transaction is opened, and then it attempts to delete the artifacts themselves when it is committed.
      Abandoning this type of transaction would re-insert the opaque records for any artifacts that haven’t been deleted, but (unlike most other transactions) a failed commit is not reversible.
    * A description of this is being worked on in [DMTN-249](https://dmtn-249.lsst.io).
      Known issues:
        * Assumes we can get URIs signed for delete, which is not true.
        * We may want to add support for saving some transaction headers directly to DB (doesn’t work for workspaces).
        * Signed URLs don’t need to be included in records if they’re always obtained just before `Datastore.get` is called.

### More Datastore design considerations

* If `Datastore` doesn’t run opaque-table delete queries itself anymore, how do we handle safe deletion of multi-file datasets, like DECam raws?
    * Maybe we don’t, and require direct ingests of those?
      Probably breaks DECam-based CI, but maybe not any production repos.
    * Or maybe accept any kind of ingest, but always refuse to delete them?
    * Can we find a way to express what we’re doing now through the new ABCs, e.g. by having `OpaqueRecordSet` have a `delete_from` method that takes a `Database` object and runs the direct query itself?
* `OpaqueRecordSet`
* Can we enshrine the categories of `Datastore` storage, for (at least) transfers and ingest?
  Propose files, opaque records, and JSON-compatible dicts.
    * This is at least helpful, maybe necessary for unifying our proliferation of transfer/import/ingest implementations (prompt processing in particular is having a tough time with this).
    * Relevant for client-server because transfers from local datastores to central datastores gets very important
    * Resolve "are `Formatters` an implementation detail of `FileDatastore` or a `Butler` public API component?"
* What even is a `SasquatchDatastore`?
    * Even if we implemented `get()`, it wouldn’t own the things it puts (it can’t delete them itself, or even keep them from being deleted).
    * But wait, this is what `FileDatastore` does with artifacts ingested with `transfer=None`.
      By putting opaque records for these into the database, we implicitly declare that these artifacts may not be deleted.
      But repository invariants rely on this!
    * Can/should we declare the same for `SasquatchDatastore`  artifacts, and start inserting opaque records or `dataset_location` entries for those?
      Doesn’t matter as long as we don’t support `put()`.
    * Should we not be inserting records for `FileDatastore` direct ingests either?
      Feels like a bigger conceptual problem than a practical one.
    * If we’re okay with what `FileDatastore` does with "direct", this is all moot unless we add get to `SasquatchDatastore`.
    * Alternative would be to define "internal datasets" vs "external datasets" with different consistency models and "is this dataset stored" logic.
      I don’t think we can have "internal datastores" and "external datastores", because `ChainedDatastore` can’t be one or the other.
* `Datastore` configuration should be repository-controlled.
    * Are there some bits of datastore configuration that we need to permit to be per-client?
      If so, can we wall those off somehow?
    * If datastore artifacts are only written inside artifact transactions, are there some bits of datastore configuration that are per-transaction, with repository config providing defaults and client that opens the transaction able to override?
    * Should we dump (some, or a summary of) datastore configuration to the database (like we do with dimension configuration) to guard against changes?

### Cleanup work

* Pydantic All The Things
    * `daf_relation`
    * `DimensionRecord`
    * `DatasetType`
    * `DimensionGraph`
    * containers of `DataCoordinate` and `DatasetRef`
    * `Config`
* Let’s drop `InMemoryDatastore` ([RFC-965](https://jira.lsstcorp.org/browse/RFC-965))!
* Let’s drop the datastore/registry bridge types and drop `dataset_location` and `dataset_location_trash`.
* Let’s fix our package/module structure ([DM-41043](https://jira.lsstcorp.org/browse/DM-41043))
  This can be done during the meeting.
  (Can we close any `daf_butler` tickets we have open this week?)
  \[there are some old PRs that we can consider closing and Andy’s experimental PR for this new work]
    * Flatten out `core` and `core/datasets`.
    * Add `datastore` subpackage for `Datastore` ABC, move things that are only used by `Datastore` there (base `Formatter`, base `StorageClassDelegate`, file descriptor, etc).
    * Merge `SqlRegistry` into `Registry` and drop registries subpackage (git will remember anything in `RemoteRegistry` and we want to move to `RemoteButler` when we stand it up.
    * Move classes out of registry if they need to appear in the `Butler` class’s signatures or base-class implementations (or `Datastore`).
* How do we stop our tests from blocking us (`runPutGetTest` is quite complex but see [DM-42934](https://jira.lsstcorp.org/browse/DM-42934))?

### The infamous new query interface

* Jim Bosch did a lot of prototyping on this in the original [DMTN-249](https://dmtn-249.lsst.io) {cite:p}`DMTN-249` even though it was mostly unrelated to the main goal of that technote.
* There’s a not-obvious piece of work that needs to happen in order to implement that interface in `DirectButler` and implement `RemoteButler` query-result objects with method-chaining: a new `daf_relation`  engine whose leaf relations represent relational concepts that are a bit higher-level than the SQL tables, like "a dataset subquery".
    * This lets a `RemoteButler` client assemble a relation tree without having SQLAlchemy objects that should only live on the server, and then send that tree to the server to be expanded into a SQL-engine relation tree.
    * It’s also necessary for the new query interface, because it allows the modifier methods on the query-result objects to do things like, "replace this dataset subquery with one that returns the ingest_date".
      At present this would be near-impossible because the relation tree maps directly to the SQL tables and that makes the "dataset subquery" a subtree that’s with manager-implementation-dependent structure.
* The interface sketched out in the [DMTN-249](https://dmtn-249.lsst.io) prototyping directory would benefit from a broad review before we dive too deeply into implementing it.

## Work packages (MVP)

Minimal Viable Product (MVP) definition: a RemoteButler (client and server) that can do:

* `queryDatasetTypes`, `queryCollections`, and `queryDatasets` equivalents via `Butler.query`;
* `Butler.get`;
* without breaking anything in `DirectButler`.

### Cleanup work (A)

1. Remove `InMemoryDatastore`. Blocked only by [RFC-965](https://jira.lsstcorp.org/browse/RFC-965).  (David) [DM-41048](https://jira.lsstcorp.org/browse/DM-41048)
    * Have to worry about the `StorageClassDelegate` tests.
      Move `InMemoryDatasetHandle` from `pipe_base`?
2. Subpackage reorganization.
   Just blocked/blocking concurrent tickets.  (Party)

### Datastore records and the new consistency model (B)

1. Adding records to `DatasetRef`.  [DM-40053](https://jira.lsstcorp.org/browse/DM-40053), underway.  Unblocked.  (Andy)
    1. Add `OpaqueRecordSet`. \[We have decided for now that `OpaqueRecordSet` may not be necessary, instead to make `DatasetRef.datastore_records` private]
    2. Make `Datastore.get()` use records-on-`DatasetRef` when they’re already there. \[Already implemented on a branch]
    3. Make `Datastore`’s bridge attribute optional (required only for operations we’re not implementing in `RemoteButler`).

### Primitives \(C)

1. Add `DimensionRecord` container.  (Jim) [DM-41113](https://jira.lsstcorp.org/browse/DM-41113)
2. Add `DatasetType` container.  (Tim) [DM-41079](https://jira.lsstcorp.org/browse/DM-41079)
3. Add `Collection` container.  (Tim) [DM-41080](https://jira.lsstcorp.org/browse/DM-41080)
4. Add `DataCoordinate` containers.  Depends on C1.  (Jim) [DM-41114](https://jira.lsstcorp.org/browse/DM-41114)
5. Add `DatasetRef` container classes.  Depends on C2?, C4.  (?)  [DM-41115](https://jira.lsstcorp.org/browse/DM-41115)

### Butler client base class (D)

1. Do the split into ABC `Butler` and concrete `DirectButler`.
   Unblocked but disruptive to concurrent tickets.  (Tim or Andy) [DM-41116](https://jira.lsstcorp.org/browse/DM-41116)

    * Start with Butler ABC defining a minimal interface.
    * Move methods to it as we support them in both `DirectButler` and `RemoteButler`?
    * Needs `Butler.__new__` and `Butler.makeRepo` trampoline.
    * Needs work on config types (not pydantic-ification).

2. Move registry caching here.  Depends on C1, C2, C3, D1.  (Andy) [DM-41117](https://jira.lsstcorp.org/browse/DM-41117) (partial  )
3. Add Butler methods for dataset type and collection manipulation (read-only).  Depends on D2.  ([DM-41169](https://jira.lsstcorp.org/browse/DM-41169))
    * Change the `butler.registry` proxy to use these as appropriate, too.
4. Merge `Registry` and `SqlRegistry` (Andy) ([DM-41235](https://jira.lsstcorp.org/browse/DM-41235)).

### Query system (E)

1. Make `daf_relation` objects serializable.  (David) [DM-41157](https://jira.lsstcorp.org/browse/DM-41157)
2. Add level of indirection in relation trees: make leaf relations more abstract, less SQL.  (Jim or Andy) [DM-41158](https://jira.lsstcorp.org/browse/DM-41158)
3. Add `DirectButler.query` itself and its result objects (and extend as needed).  Depends on E2.  (Jim or Andy) [DM-41159](https://jira.lsstcorp.org/browse/DM-41159)
4. Make `Query` able to expand datastore records in returned `DatasetRefs`.  Depends on E3.  (Jim or Andy) [DM-41160](https://jira.lsstcorp.org/browse/DM-41160)

### RemoteButler Client/Server (F)

1. Add initial RemoteButler client and server with just initialization (David, Tim). [DM-41162](https://jira.lsstcorp.org/browse/DM-41162)
2. Add initial deployment of butler server (David).
3. Add cached-based (dataset type and collection) queries.  Depends on D3. (?)
4. Add `RemoteButler.query`.  Depends on E1, E4. (?)
5. Add `RemoteButler.get` (or maybe this is a `Butler` base class method?).  Depends on F2 and B1. (?)

## Non-MVP Work Packages

### Technotes/Design Docs (G)

1. Update [DMTN-249](https://dmtn-249.lsst.io) to talk about artifact transactions, shrink the prototype [DM-40811](https://jira.lsstcorp.org/browse/DM-40811) (Jim)
2. Update [DMTN-271](https://dmtn-271.lsst.io) to reflect [DMTN-249](https://dmtn-249.lsstio) prototyping ([DM-40641](https://jira.lsstcorp.org/browse/DM-40641) is still in review; Jim)
3. Update [DMTN-242](https://dmtn-242.lsst.io) to reflect MVP plan (Tim)

### Pipeline Workspaces / Provenance (H)

1. Create new QG classes that can represent status, avoid `Quantum` class’s `NamedKeyMapping` problem.
2. Implement QG and Pipeline expression language parsers.
3. Stand up `PipelineWorkspace` MVP
    1. Can be internal (transaction opens on creation) or external (transaction opens on commit).
    2. Can do single-process execution only (extensible to multiprocessing).
    3. Single-shot QG generation only.
4. Implement simple-multiprocessing `PipelineWorkspace`.
5. Design `Butler` system for provenance QG storage.
6. Make BPS work with workspaces (maybe even present as a Workspace subclass?)
7. Drop `pipetask run`.
8. Add graph-aware in-memory caching system for cases where multiple quanta are going to be processed in a single process (eg prompt processing).
   This is the replacement for InMemoryDatastore.
    1. Understand which datasets are going to be needed downstream and keep them in memory automatically.
    2. Understand how many times they are needed downstream (thereby deciding whether to return a copy or whether to delete from cache on get)
    3. Consider a mode where intermediates can be configured never to be persisted (prompt processing might not want to lose a second whilst an intermediate is written to disk)

### Artifact Transactions / Opaque Record Responsibility (I)

1. Reimplement `Datastore.knows` to not use `dataset_location` but to use datastore records.
    * Requires that every datastore that supports `get()` stores something in the records table.
2. Make dataset deletion not use `dataset_location[_trash]`
    * Does `Datastore.emptyTrash` go away?
    * Use persisted transaction file containing all the files we are going to delete?
      Then delete them?
3. Create migration to get rid of `dataset_location[_trash]` and maybe update opaque table FK details.
4. Remove bridge and access to opaque records from `Datastore`.
5. Add `ArtifactTransaction` ABC and supporting methods to `DirectButler`.
    * At this point artifact transactions are allowed as a way to do modifications, but are not required.
6. Reimplement all database plus storage write operations in terms of artifact transactions.
    * This is easiest if we can drop support for non-workspace execution beforehand.

### Import/Export/Transfer/Ingest overhaul (J)

1. Add registry "batch edit" concept.
2. Reimplement other `DirectButler` database-only mutations in terms of registry batches.
3. Design import/export/transfer/ingest overhaul using registry batches and artifact transactions.

### Butler Client Server Phase II (K)

1. Add authentication immediately following MVP. Understanding how this affects the APIs is important.
    1. Add new tables (hence schema migration) to support ACLs.
    2. Can we add permissions model to `DirectButler`, too?  (not rigorously, but as guard against accidents)
2. User authorization: read-only collection filtering would be a good test of [DMTN-182](https://dmtn-182.lsst.io) {cite:p}`DMTN-182`
    1. Only pass allowed collections to client cache.
    2. Still need to check permissions on butler.get because we do not trust anyone.
    3. Can a user make one of their RUN collections part of a group?
    4. If not `u/` or `g/` then public read-only by default.
    5. Group management allows read/write control.
    6. Tech note talks about ACLs?
       Is that how we allow user run to be visible by other user?
    7. Will we have to provide a way to mv datasets from run to run, whilst preserving provenance? (i.e., retaining the UUIDs when we move them).
3. Switch to using signed URLs in butler.get()
    1. `DatasetRef` has the relative path in the record.
    2. Client passes this information back to server for signing.
       Server checks permission for accessing relative path and returns signed URL without requiring database lookup.
       Absolute URIs are always allowed and converted to signed (but we have to check that someone hasn’t prepended datastore root to their relative URI to bypass permissions check).
4. Support butler.getURI in client
    1. Returns server URIs not signed URIs.
5. More advanced query support: pagination.
    1. Maybe implement default limit of 5000 results and then paginate if people explicitly ask for more.
    2. Deduplicate in server?
       With 100k returns you need to keep them all in memory.
    3. Would need to enable SQLAlchemy lazy returns if not deduplicating and if streaming the results directly to pagination files.
6. `Butler.transfer_from()` remote to local? (implemented in workspace already?)
7. `Butler.put()` implementation.
8. `Butler.ingest()`
    1. What about “direct” mode?
       This surely can’t work unless the files being ingested are also visible to the server (so likely https direct).
       Allow but require the server to check to see if they are visible?
9. `Butler.transfer_from` local butler to remote butler.
    1. `Datastore` records now in registry at both ends and not in datastore.
    2. Artifact Transactions?
       How to handle chained datastores at both ends where N datastores are transferring to M datastores.
10. Butler export/import
    1. Do we rationalize import/export such that it uses JSON Pydantic serializations as we already use in client/server?
11. Materialization on the client (as python data structure sent back to server) for graph building efficiency.
    1. The client knows it can read the pipeline definitions and python classes whereas the server only depends on `daf_butler` and does not depend on science pipelines docker container.
       We therefore can’t easily have a server side graph builder.
    2. We mimic server-side temporary tables by uploading a table stored on the client every time a query against it is executed.
    3. If this is too slow we could potentially store tables as files on the server, or even use non-temporary tables whose lifetimes we track some other way, but this all gets pretty messy.
       Moving more QG generation joins to client may be a better option, even if it means fetching more than we need from the DB up front.
12. Can client server support `butler ingest-raws`?
    This is tricky in that it requires that dimension records need to be created for this.
    Do we trust people to create dimension records?
13. When can client server support `registerDatasetType`?
    Should it ever?
    For user-namespaced only?
14. Is `syncDimensionData` ever allowed in client/server?


## QuantumGraph Provenance Storage (L)

1. Store quantum graph files as real but private datasets.

    * Managed by database and datastore in the usual way.
    * Dataset type prefix is just configured in the butler config, base `Butler` implementation just reads it (with parameters/components) as needed.
      Different dataset types for different sharding dimensions.

1. Store quantum graph files as files managed directly by butler.

    * `Butler` asks `Datastore` for a root (same as with `ArtifactTransaction` space).
    * We save the names of special provenance dataset types with each RUN.
        * These could be asked to annotate the QG with extra information that `daf_butler` doesn’t know about (e.g. `PipelineGraphs`).
          This would happen on the client in `RemoteButler`.
    * Butler just calls some QG Python method to read/write.

1. Store quantum graph files via a new polymorphic configured Manager type

    * Still saves everything exactly as in the previous options under the hood.
    * Can put this in the migration version system.
    * Might be weird to have a Manager of a Butler, not a Manager of a Registry, but maybe all managers will go that way someday.

* “Simple” provenance storage in dataset output files to avoid querying provenance at all.
  This might be prudent on DR1 release day to avoid everyone asking what raws went into a coadd.
    * Make sure that Exposures include all the input DatasetIDs.
    * Really could do with “raw” observation_ids of all inputs stored in Exposure as well to quickly answer question of what were all the input raws.
      The graph builder has to propagate this information through from the beginning.
      Could be UUID instead of OBSID if we want to be completely generic about choosing primary root dataset type.



## Blue Sky Thinking

1. Create new `Datastore` that returns records containing a representation of the content of the in-memory dataset.
    1. Each `StorageClass` gets its own schema defining the columns that are returned.
       Or else define the schema and the implementation that returns the table data in a `StorageClassDelegate` class.
    2. Once we do this we can include these (seeing etc) in queries.
    3. How does schema migration work if someone adds a new field to the `StorageClass`?
    4. Do we have to add a “name” to the `StorageClass` definition that is used to name the records table?
       Or do we name them after the `StorageClass`.
    5. Is it one datastore per storage class? Or one datastore that always tries to lookup the delegate and call the corresponding delegate method?
2. Allow Datastore records to be included to constrain queries.
    1. `WHERE datastore.file_size > 10_000_000` (need to use the datastore name here?)
3. Make `ingest_date` more public and visible outside of `WHERE` clauses?
4. Move `pipetask` to `pipe_base`.
    * `pipetask build` and `pipetask qgraph` only need `pipe_base`.
      `pipetask run` is the thing that needs `ctrl_mpexec`.

## Questions for Russ Allbery

Russ Allbery was invited to call into the meeting to answer questions that had arisen up to that point.

* How early and how often do we sign URIs, especially for get?
    * If signed URIs time out really quickly, there’s no point returning signed URIs in registry queries that return `DatasetRef`s.
      We always have to sign URIs right in `Butler.get()`  and `DeferredDatasetHandle.get()`.
    * If signed URIs time out more slowly and it’s much faster to do a lot together than one-at-a-time, we should sign up front in those registry queries.
    * Can the client tell whether a signed URIs has timed out without talking to a server, in order to know whether to refresh it before it actually attempts to use it?
    * Are there any options for client-side signing that we should consider?
        * Answers:
            * Signing URLs is entirely in server but there is some overhead.
            * May not want to sign 100k of them in server without knowing that the client really is going to use them.
            * The timeout can be set to anything up to 7 days.
              Probably don’t want longer than 8 hours for graphs.
            * Consider returning the expiry time of the signed URL with the query results.
            * May need to have query API that lets user opt-in to getting datastore records.
              No reason to do it if they don’t need them.
            * No way to delegate credentials to a client to do its own signing.
              The signed URL is for a specific key in a specific bucket.
            * Signed URL requests can be fast.
              Opaque token?
              If we aren’t shipping people signed URIs we send them the datastore "path-in-store" and then send that back to server for signing.
              The server then checks to make sure they have permission to read that key.
            * Signed URLs can not be used for deletions which means that `Butler.pruneDatasets` can’t be deleting things using the client datastore itself.
* Pydantic validation contexts
    * This is a thing that gets passed to custom Pydantic deserializers (in Pydantic v2), and it seems like a way to get them access to state (e.g. `DimensionUniverse`) that is global for the server but not global in any other context.
      Is there a way to get FastAPI to do this when it’s what’s invoking Pydantic deserialization.
        * Answers:
            * Global state on the server (set to `None` in the client, set to not-`None` in server startup).
            * When `None` (on client, in `DirectButler`), use Pydantic validation context.
* Pydantic patterns for polymorphism.
    * Answers:
        * When using discriminated unions, avoid "extra fields" in models.
* Server APIs
    * Have some way to check that transactions closed.
    * Have some way to do some consistency checks.
    * Server not async so will need to horizontally scale very early on.
      Will use thread pool by default if sync.
    * Concerned about no async since threading may cause some unexpected behavior.
    * On the other hand async-everywhere in server may need a lot of work and might cause its own problems.
    * User name comes in automatically in the HTTP header.
    * Groups-of-user use `Depends` scheme in FastAPI.
    * If we have a butler client/server MVP we can immediately put that into Mobu [SQR-080](https://sqr-080.lsst.io) {cite:p}`SQR-080` for scaling (does not need to be a notebook for this).

* Best practices for versioning -- method-by-method, or does a server declare a set of supported versions that all methods satisfy?
    * Slightly easier to v1 → v2 everything.
      Batch up API changes all in one go rather than changing versions piece meal.
    * Upgrade server first then release new client and drop v1 when you see that v1 API usage drops below threshold.
    * New clients talking to old servers is a pain.
      Tell people to use the older clients.
    * Alternative: put version number in HTTP header instead.
      GitHub use mime type to control which API a client is using.
      This is more work.
      URL with vN in it is easier to look for in logs.
    * Avoid clients trying to support many different versions.
      May not have a choice.
    * Need to consider adding un-versioned server API for returning supported versions.
      That can be problematic in itself since it effectively forces you to use global v1 → v2 migrations.
    * Modern systems use a capabilities approach (e.g., TAP).
      Try to avoid having to publish every single endpoint with versions.
      Could envisage having a capabilities approach that says put is not supported but get is.
    * The server will become a public interface.
* Refresh on [DMTN-182](https://dmtn-182.lsst.io); how sophisticated is the permissions model?
    * Does Butler need to store ACLs?
      (Yes, we think.)
    * ACL is for user who creates a private run but then realizes they want to sure.
    * Alternative is to move RUN into a group collection.
      Then no need for ACL at all (maybe).
      RUN moving is hard at present.
    * Person X must be member of group before they can add an ACL of a private RUN to that group. Butler can not create groups -— groups are handled centrally.
    * In theory separate read/write ACL is possible.
    * Groups do not have ownership.
      Any member of group can delete.
    * Do not trust people with deletion abilities -— people make mistakes and get confused.
      Best if deletions delete virtually in registry first but don’t really delete for 30 days.
    * ACLs should be read-only access.
      Read/write should only be in actual groups. ACL can also allow public read.
    * `butler` command line tool for setting and removing ACL using the REST API.
       Javascript front end later on.
        * Maybe allow “path” prefixes rather than arbitrary prefixes.
          Split on “/”.
        * Russ promises web page to list group membership.
          Command line can poll the group API as well to get list up front.
* The JSON example: `u/timj` vs `g/timj` but what happens if `timj` makes `g/jbosch` and then `jbosch` turns up later?
    * `g/timj` is the user group but `g/g_timj` is a user-created group.
      So there is no problem with clashing.
    * Group conflicts with user name is not our problem.
    * At SLAC RSP the groups do not match US-DAC groups.
      At SLAC will need to call `getrgroups`.
    * Would be nice if `DirectButler` also checked collection names for users.
* Tend to agree that users should not be able to even see lists of collections that they do not have permission to see.


## References

```{bibliography}
  :style: lsst_aa
```
