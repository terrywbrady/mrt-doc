# 2018-09-25: Simplifying Merritt

See:

- [preparatory notes](2018-09-24-simplifying-merritt-prep.md)
- [whiteboard / meeting notes](https://docs.google.com/document/d/1-45bYKxiyDlJx5LrJIbMdPzPYXX8ALon6sQDVspIJLM/edit) (Google Docs)

## Issues

### Reactive process

We want to be able to concentrate on tasks rather than be constantly
reacting to crises & service problems.

We also have a problem with jumping onto tasks prematurely because
they happen to be a topic of conversation, or because some outside
stakeholder is suddenly ready for us or suddenly taking an interest.

#### Action items / questions to answer

1. crises
   - what can we learn from past crises?
   - what can we do to reduce the frequency of crises?
   - do we have a clear sense of what's a crisis and what's not?
   - what can we do to handle crises better when they arise, to minimize
     impact on the team?
2. lack of focus
   - are we setting priorities clearly?
   - what causes us to jump on new tasks out of order?
   - who should be managing the schedule / gatekeeping outside requests?
   - are there bigger questions we need to answer about how we decide what
     to work on?

### Service management

#### Deployment

We want to be able to add/remove/redeploy any instance of any service at
will. Obstacles to that include:

- complexity of deployment/configuration
  - configuration of individual servers (e.g. repository/node directories on storage)
  - configuration of communication between services (mostly load-balanced, sometimes server-locked)
    - what's server-locked (in configuration) besides UNM? UCSB?
    - note that we can probably remove UNM storage early next year
  - per-server Capistrano environment configurations
- complexity of managing load balancers (ALB, ELB, Apache)
- complexity of startup process
  - order dependence of services (everything depends on storage)
- fragility of long-running processes
- difficulty of determining current state of the system
- lack of orderly shutdown process (no "closing" state that can finish
  existing work without accepting new work)

Note that these same issues also make it difficult or impossible for us to
move to automatic patching / restarts.

##### Action items / questions to answer

Some long-term goals:

- [blue/green deployment](https://martinfowler.com/bliki/BlueGreenDeployment.html)
- [continuous delivery](https://martinfowler.com/bliki/ContinuousDelivery.html)

Short- to medium-term action items (as much as possible before move to Linux 2):

1. for each service, figure out what's involved in setting up a new instance
   & make a checklist (to some extent these may already exist)
   - system configuration, packages, etc.
   - additional software installation (Tomcat?)
   - Merritt software
     - build
     - deployment
     - configuration
   - Apache and/or ALB
2. figure out *how* we want to add/remove/redeploy services (ideally)
   - some combination of Puppet and Capistrano?
   - target audience: Jim? Mark? Perry?
3. identify places where we assume a certain fixed set of servers
   - load balancers
   - Capistrano environments
   - UNM storage nodes (note: we can probably remove these early next year)
   - ???
4. for each service and server, make sure all (production) configuration is
   under source control
5. for each service:
   - identify obstacles to fully automated setup
   - make a plan to remove those obstacles

(One idea, per Jim: a small 'bootstrap' instance with one of every service)

Longer term action items:

- introduce "closing" state for orderly shutdown
  - may require coordination between services & load balancers?
- when services are unavailable, time out & retry gracefully
- simplify state determination
- figure out how not to lock ingest jobs to servers (more on that below)

Q: Do we need a 'starting' state as well as a 'closing' state?

(Note: we should figure out which of these are either lowest-hanging fruit or most
bang for the buck, and prioritize those)

#### Complexity

Apache

- 65 distinct rewrite rules
- multiple roles: load balancer, front end, URL munger...?
- leads to double or triple encoding/unencoding

LDAP

- authentication (username/password)
- authorization (user permissions by collection)
- user profile metadata (name, phone, etc.)
- UI (`mrt-dashboard`) code is inconsistent in its use of LDAP vs. database for collections

##### Action items / questions to answer

Apache:

- are there any rewrite rules we can already remove?
- is anything going through Apache that doesn't need to?
- what are the obstacles to eliminating Apache entirely?

LDAP:

- is there an auth/auth solution that:
  1. can be based on the inv database
  2. can work with Rails (ideally with Devise, cf. DMPTool)
  3. can replace LDAP auth/auth in `mrt-sword` and `mrtexpress`?
- if `mrt-sword` and `mrtexpress` already talk to the inv database,
  maybe we can move the authorization piece and keep LDAP just for
  authentication, as a first step

### Local copies

Making local copies limits the size of objects we can handle, introduces
complexity (e.g. locking ingest jobs to servers), probably increases costs
(storage on each server), and may reduce performance (at least in some
scenarios).

#### Action items / questions to answer

Questions:

- In the past we've said we have to make local copies because services / the network are
  unreliable.
  - just how unreliable are they? can we quantify that?
  - where can we mitigate w/retries etc., and where is that not possible?

Straw proposal:

- Ingest
  - stream uploads directly to S3 (even for collections where S3 is not primary)
  - return S3 URLs instead of ingest server URLs
- Storage
  - uploads:
    - abstract move/copy/store operation across different combinations of storage:
      - S3 -> same S3 bucket: no-op
      - S3 -> different S3 bucket: use s3 sync, verify, delete
      - S3 -> OpenStack: stream instead of making local copy
        - maybe use [rclone](https://rclone.org/)?
      - S3 -> Cloudhost: stream instead of making local copy
        - create [pre-signed URL](https://docs.aws.amazon.com/AmazonS3/latest/dev/ShareObjectPreSignedURL.html)
          & pull from Cloudhost side?
    - downloads:
      - Use `ZipOutputStream` to stream large zip files directly to network
- Audit
  - stream directly from storage
- Replication
  - from S3: see above under storage/uploads
  - OpenStack -> S3: stream instead of making local copy
    - maybe use [rclone](https://rclone.org/)?
  - Cloudhost -> S3: just don't support it?

Alternatives / other areas to explore:

- use [s3fs-fuse](https://github.com/s3fs-fuse/s3fs-fuse/wiki/Fuse-Over-Amazon)
  to provide an S3-backed "local" filesystem
- look at [Dat](http://datproject.org/) and/or other alternatives to straight HTTP
  for large file transfers

Are there other issues unique to each service that need to be addressed?

### Scale

We want to be able to handle very very large objects -- currently 500 GB is
hard, 1 TB can't be downloaded; 10 TB is nearly impossible. We also want to
be able to handle the volume of requests/submissions we'd get if we could
provide libraries with free storage.

Issues include:

- factors out of our control on the submitter's side
- local copies for ingest, storage, audit & replication
- lack of scalability testing / load testing
- ????

#### Action items / questions to answer

- don't make local copies
- look at [Dat](http://datproject.org/) and/or other alternatives to straight HTTP
  for large file transfers
- support incremental deposit w/o "versioning"
- scalability/load testing (UI, Ingest, …?)
- other issues?

### Environments / Testing

We don't need to support both "dev" and "stage" environments. what we need is:

1. one QA environment, as realistic as possible
2. a certain number of machines (VM) for exploratory/experimental development
3. a safe way to test against production content, probably read-only

#### Action items / questions to answer

environments: immediate

1. create new Dash demo collection in production, delete old one from
   stage (need to coordinate w/Dash)
2. identify which machines with a `-dev` suffix are not really part of the
   dev environment (`uc3-mrtreplic1-dev`, `uc3-mrtdat1-dev` ... others?)
3. decomission the other `-dev` machines (need to coordinate w/Dash)

environments: medium-term / Linux2 migration

1. inventory current `-stg` and `-prd` environments, as well as "not
   really dev" `-dev` machines above
2. plan new Linux2 environments:
   - `-prd`: new production environment
   - `-stg`: new QA environment, based on `-prd`
     - plus: Cloudhost native & Docker, at least pro tem
   - `-dev`: non-"environment" experimental/test machine(s); new name(s)?
3. stand up new `-dev` machine(s)
4. shut down remaining old `-dev` machines
5. stand up new `-stg` environment to prove out Linux2
   - possibly rotating new Linux2 machines into load balancer etc. to simulate
     production migration
6. stand up new `-prd` environment
   - rotating Linux2 new machines into load balancer etc.
7. shut down old `-stg` and `-prd` environments (need to coordiante w/Dash)

production-scale read-only testing

- figure out what kind of tests we need to run, and what environments we
  need for those tests (is this our `-stg` environment, or our experimental
  environments?)
- talk to IAS about Amazon options for limiting S3 access to read-only
  for certain boxes
- talk to SDSC about read-only credentials for OpenStack access
- other concerns?

production-scale write testing

- figure out what we can learn from either small numbers of large
  objects, or large numbers of small objects, either of which we should
  be able to afford
- make sure servers and writable S3 test bucket are in the same region
  so we don't have data transfer charges
- other concerns?

### Administration

We want it to be easy for customers to monitor the progress of their ingest
jobs and/or be notified when problems occur, and we don't want Perry to
have to keep an eye on the ingest pipeline himself; we also want better
tools for tracing jobs through the workflow, and some sort of an
administrative dashboard.

#### Action items / questions to answer

- what are the limitations of our current ability to trace jobs through the
  workflow? do we need to attach more metadata, job IDs, etc. somehow?
- how can we improve notifications?
- how can we improve workflow visibility?
- what features do we want in a dashboard?

### Collection management

We want to be able to move objects easily between collections. Users,
especially libraries, want better tools for organizing their objects into
collections -- larger numbers of collections, hierarchical collections,
etc.

Meanwhile, creating and administering collections is more trouble than it should
be. Information about collections is distributed between LDAP, the
inventory database, and ingest profile files.

#### Action items / questions to answer

Simplifying collection administration:

- stop using LDAP for authorization (see above under [Complexity](#complexity))
- simplify ingest (proposal) :
  - pass collection mnemonic or ARK to ingest instead of profile
  - use the database instead of LDAP to identify collection "ingest profile"
    (but see below)
  - get storage node info from inv database
  - move notification email into inv database
  - separate out ARK minting configuration (between shoulder, & minter URL,
    probably < 10 different configurations)
  - separate out ingest handling (probably < 5 different configurations)
  - use DB to identify ARK minting & ingest handling "profile"
- give Perry some better tools for creating/modifying collections

Supporting object moves & different collection management scenarios:

- obstacles to moving objects between collections?
- obstacles to large numbers of collections?
- obstacles to hierarchical collections?
  - conceptual problems
  - UX problems for ordinary users
  - UX problems for administrators
  - special curatorial UX?
- in light of the above, consider separating "collection" into some/all of:
  - curatorial collection
  - access control context
  - ARK minting configuration
  - storage/replication configuration

### Cost

Cost is a barrier to Merritt adoption for campuses, and particularly for
libraries, leading to bad preservation decisions.

We've seen some outside interest in providing free storage for open
research data but little to none for library data, esp. dark archives.

#### Action items / questions to answer

- why are outside providers interested in free storage for research data, but
  not for libraries?
- what can we do to bring on-line storage costs down?
  - SDSC Qumulo / Minio: ca. $70/TB/year (projected)
  - [Backblaze B2](https://www.backblaze.com/b2/cloud-storage-pricing.html): $60/TB/year
- what can we do to bring dark-archive storage costs down?
  - [Glacier](https://aws.amazon.com/glacier/pricing/): $48/TB/year
  - [IBM Regional Cold Vault](https://www.ibm.com/cloud-computing/bluemix/pricing-object-storage): $72/TB/year
  - [Oracle Archive Storage](https://cloud.oracle.com/storage/pricing): $32/TB/year
  - SDSC Qumulo / Minio: ca. $70/TB/year (projected)

### Miscellaneous

#### Action items / questions to answer

Ingest:

- if we stop making local copies, what else do we need to do to stop locking
  jobs to ingest servers?
- what would we need to do to ingest and inventory to support ingesting multiple
  versions of the same object out of order? (simplifying assumption: the start of
  each ingest job is ordered correctly by version, it's the finish that isn't,
  e.g. when the first version is large and subsequent versions are small)

Storage:

- how can we mitigate the complexity of multiple storage technologies?
  - what are the areas in which supporting multiple technologies becomes a
    problem?
  - can we reduce the number of storage technologies we support?
- what do we need to do to be able to move content w/o re-ingesting?

Other:

- modernize logging / make logging consistent
  - figure out what this would mean
- can we move Dash/Dryad to a Merritt Express–based model cf. eScholarship?
  - investigate generating [pre-signed URL](https://docs.aws.amazon.com/AmazonS3/latest/dev/ShareObjectPreSignedURL.html)s to give clients direct access to S3 so it doesn't have to be streamed through another server
