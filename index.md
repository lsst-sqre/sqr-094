# TAP Query History via UWS

```{abstract}
A requirement for the RSP and in particular the portal is providing users the ability to view their past queries and possibly interact with them, for example by re-running, storing or sharing with others.
This note describes a proposed approach to address this functionality via the use of the “jobs” API  which is part of the UWS standard and available via the CADC TAP service.
```

A requirement for the RSP and in particular the portal is providing users the ability to view their past queries and possibly interact with them, for example by re-running, storing or sharing with others. This note describes a proposed approach to address this functionality via the use of the "jobs" API which is part of the UWS standard and available via the CADC TAP service.

## 1. Introduction

The recommended approach in this note describes how it is possible with existing features of the TAP service that the RSP already operates to get metadata on queries run by a given user. There are limitations to this approach which are described in this note, and so are possible mitigation strategies, though more details on the severity of the limitations or the need for mitigations may be further analyzed at a later point. Some of the goals achieved through this approach include:

- Minimize effort required to get a query history MVP for TAP
- Reuse existing standards & solutions
- Ensure that non-MVP use cases are still possible with approach
- Ensure Firefly is interacting with a VO standard to retain its generic nature

## 2. Relevant Standards and Terminology

### 2.1 Relevant Standards

Universal Worker service pattern 1.1: https://www.ivoa.net/documents/UWS/

### 2.2 Terminology

- UWS - Universal Worker service pattern
- Jobs & queries are used interchangeably in the context of this technote.
- "{jobs}" refers to the UWS async jobs endpoint which can be found at the base url of the TAP Async endpoint, e.g. https://data.lsst.cloud/api/tap/async

## 3. Proposed Architecture

### 3.1 Overview

The RSP TAP services implemented using the CADC TAP software all get the /{jobs} UWS endpoint out of the box, which will be used by clients such as Firefly to obtain the metadata needed to construct the query history. No additional services will be written at this point, and at the point of writing there is no obvious need for changes to either phalanx or any of the other Rubin applications such as Gafaelfawr.

A very general overview of how Firefly interacts with TAP, and how TAP then fetches content from the UWS database is shown below.

```{figure} sys-diagram.png
:figclass: technote-wide-content

System diagram
```

### 3.2 Access Scenarios

There are at least two possible access scenarios that Firefly & potentially other clients may want to use to get the query list from UWS.

#### Option 1: Get list of queries (including query text)

In this scenario, Firefly retrieves the user's list of queries from the {jobs} endpoint, and for each query makes an additional request to the job detail endpoint to get additional metadata, including the query string itself. It then displays this list to the user in whatever way is decided on the client's end.

```{figure} flowchart1.png
:figclass: technote-wide-content

Flowchart (inc. query text)
```


#### Option 2: Get list of queries (not including query text)

In this scenario, Firefly retrieves the user's list of queries from the {jobs} endpoint, and displays this list to the user. An additional user action may trigger Firefly to request additional metadata for the query from the job detail endpoint.

In any case Firefly or other clients can request a filtered list of queries from the {jobs} endpoint. There are a few server-side filtering options (by phase, time) listed below, and other filtering may be added on the client side if needed. An example of the latter is if Firefly wants to allow users to filter by query text.

```{figure} flowchart2.png
:figclass: technote-wide-content

Flowchart (not inc. query text)
```

## 4. Implementation Details

### 4.1 The {jobs} Endpoint

The {jobs} interface can be found here: http://{base-url}/api/tap/async for the various TAP services currently deployed in the RSP. This endpoint is defined as a requirement by the UWS standard, and our TAP service (CADC) includes the endpoint as a product of their implementation.

#### API

The following describes two of the relevant API paths in UWS, limited to GET requests. These are related to fetching a list of jobs, and job specific metadata. There are additional POST paths that alter the behavior, which can be found in the IVOA UWS specification.

- **GET /async**
  - Description: Retrieves a list of all asynchronous jobs.
  - Format: Returns a list of jobs with details such as job-id, phase (PENDING, RUNNING, COMPLETED, ERROR etc.), and metadata.
  - Filtering: Supports query parameters like PHASE, AFTER, LAST

- **GET /async/{job-id}**
  - Description: Provides details for a specific job identified by job-id.
  - Format: Returns job status including phase, start and end times, and any error messages.
  - Additional Operations:
    - To abort, use POST /async/{job-id}/phase with PHASE=ABORT.
    - To delete, use DELETE /async/{job-id}.
    - Results are available at /async/{job-id}/results/{file} and parameters at /async/{job-id}/parameters.

#### Functionality and data format

As defined by the standard, this gives us a list of all queries that the given user can see. The query metadata is returned in the following format:

```xml
<uws:jobs xmlns:uws="http://www.ivoa.net/xml/UWS/v1.0" xmlns:xlink="http://www.w3.org/1999/xlink" version="1.1">
  <uws:jobref id="ae5m9odqwt7wi">
    <uws:phase>COMPLETED</uws:phase>
    <uws:ownerId>user</uws:ownerId>
    <uws:creationTime>2024-07-31T19:52:48.567Z</uws:creationTime>
  </uws:jobref>
  <uws:jobref id="afos4ifdqromqx5yk">
    <uws:phase>ERROR</uws:phase>
    <uws:ownerId>user</uws:ownerId>
    <uws:creationTime>2024-08-10T21:05:56.928Z</uws:creationTime>
  </uws:jobref>
</uws:jobs>
```

Note that there is one field that does not appear there, which based on recent discussion with CADC may be an oversight and thus be included in future releases is runId. The relevance here is it may be possible to annotate the client that initiated the query (Firefly-metadata-queries for example) in the runId column, which would then appear in the list, making it easier for Firefly to filter out certain queries.

Further metadata can be requested from the job endpoint an example of which is:

```xml
<uws:job xmlns:uws="http://www.ivoa.net/xml/UWS/v1.0" xmlns:xlink="http://www.w3.org/1999/xlink" version="1.1">
  <uws:jobId>ae5m9odqwt7wi</uws:jobId>
  <uws:runId/>
  <uws:ownerId>user</uws:ownerId>
  <uws:phase>COMPLETED</uws:phase>
  <uws:quote>2024-08-01T19:52:47.582Z</uws:quote>
  <uws:creationTime>2024-07-31T19:52:47.583Z</uws:creationTime>
  <uws:startTime>2024-07-31T19:52:48.567Z</uws:startTime>
  <uws:endTime>2024-07-31T20:01:57.857Z</uws:endTime>
  <uws:executionDuration>14400</uws:executionDuration>
  <uws:destruction>2024-08-07T19:52:47.582Z</uws:destruction>
  <uws:parameters>
    <uws:parameter id="LANG">ADQL</uws:parameter>
    <uws:parameter id="QUERY">SELECT TOP 1 objectId, coord_ra, coord_dec, psfFlux FROM dp02_dc2_catalogs.ForcedSource WHERE psfFlux BETWEEN -1500 AND -1505 ORDER BY psfFlux DESC</uws:parameter>
    <uws:parameter id="REQUEST">doQuery</uws:parameter>
  </uws:parameters>
  <uws:results>
    <uws:result id="diag" xlink:href="uws:executing:4"/>
    <uws:result id="diag" xlink:href="jndi:lookup:0"/>
    <uws:result id="diag" xlink:href="read:tap_schema:21"/>
    <uws:result id="diag" xlink:href="query:parse:2"/>
    <uws:result id="diag" xlink:href="jndi:connect:1"/>
    <uws:result id="diag" xlink:href="query:execute:548762"/>
    <uws:result id="diag" xlink:href="query:store:500"/>
    <uws:result id="rowcount" xlink:href="final:0"/>
    <uws:result id="result" xlink:href="https://data-dev.lsst.cloud/api/tap/results/result_ae5m9odqwt7wi.xml"/>
  </uws:results>
</uws:job>
```

#### Filtering options

Because the list of queries under {jobs} is likely to grow indefinitely depending on factors like query rate & destruction time UWS allows the endpoint to be filtered. The current server-side filtering options are currently available:

- **Get by Phase**
  - Example: /{jobs}?PHASE=EXECUTING
  - Returns all queries that match the given phase. Phase in the UWS standard can also be interpreted as a query status.

- **Get Last {x} queries**
  - /{jobs}?LAST=100
  - Return the last 100 queries (Ordered by creationTime)

- **Get queries after given date**
  - /{jobs}?AFTER=2014-09-10T10:01:02.000Z
  - Return all queries after the given date and time in UTC.

A client may also choose to provide filtering on any other of the job parameters. An example is filtering queries that match a certain query string. This is not possible via a query to the job list, but could still be done client-side (i.e. Firefly).

### 4.2 Job Lifecycle

#### Archived jobs

Every job in the current TAP implementation gets a "destructionTime". This is currently set to 1 week after the job start time. Upon reaching the destruction date-time the services will set the job status to "ARCHIVED" (at least this is what is defined in the standard). Archived jobs will not show up in the job list.

While this is what is defined in the standard, the CADC implementation does not actually change the "executionPhase" field in the uws.job table.

#### Job Deletion & Garbage collection

Aside from policy by which jobs become archived, the UWS standard also allows deleting jobs by a user-triggered action. This is done via an HTTP DELETE Request, though it is also possible via posting ACTION=DELETE to the job. What the service actually does with this may be subject to interpretation in the current form, as it is not very clear what the distinction is in the expected outcome of a DELETE action and the action taken at destruction time.

Initial investigation into how the CADC TAP service handles this is that it never actually changes the phase to "ARCHIVED", but there is a deletedbyuser boolean field that is set to 1 when the job is deleted via HTTP DELETE.

These jobs do not appear in the jobs list, and the assumption is (needs further confirmation) that jobs that exceed the destructionTime also do not appear in the list, but likely through a CADC filtering process that checks and compares dates before showing the job.

#### Job Deletion in Pyvo

Jobs run via Pyvo and specifically via the run_async method (https://github.com/astropy/pyvo/blob/01f2be56a7551826c6c2a3a7e459feed9da68f8a/pyvo/dal/tap.py#L283) will send an HTTP DELETE after the job is completed. Thus if running a query to the CADC TAP service via pyvo async, the job will not appear in the job list.

#### Job Deletion in Topcat

When running a query via Topcat, users will be presented with a checkbox defining whether they want to delete the job on exit, which defaults to True. Thus in most cases async jobs run via Topcat will also be deleted and not appear in the list. It's worth noting that Topcat defaults to synchronous jobs, so it's probable that most jobs will be run as sync.

### 4.3 Authentication and Authorization

Since the request to the {jobs} endpoint is coming from Firefly, or possibly in the future from a CLI or python tool, a token will be passed in to the request as with any other request to the authentication endpoints of the TAP service, which are a token specific to the user. The TAP service will thus identify the user and is able to return their queries only.

### 4.4 Query History Proof-of-Concept Python Module

A sample Python module that does query history can be found in the following link:
https://github.com/stvoutsin/queryhistory

This demonstrates how to abstract the jobs API behind a query history class, that adds some additional client side filtering functionality and may be useful for visualizing how other clients like Firefly may interact with the {jobs} endpoint.

A quick usage example of the for this module:

```python
query_history_service = QueryHistory(base_url=JOB_URL, token=TOKEN)

# Get completed queries
queries = query_history_service.get_queries(limit=5, phase="COMPLETED")

# Get last 5 queries
queries = query_history_service.get_queries(last=5)

# Get queries after August 20th
queries = query_history_service.get_queries(after=datetime(2024, 8, 20), last=3)

# Get queries that match a specific query string
filters = [
    lambda q: q.query_text == "SELECT TOP 10 * FROM ivoa.ObsCore",
]
queries = query_history_service.get_queries(limit=5, recent=True, filters=filters)
```

## 5. Alternative Approaches Considered

### 5.1 New FastAPI Query History App (Alternative Query Store)

An alternative approach that was considered to use UWS as the source of user queries and the {jobs} endpoint as the API was to set up a new FastAPI service and query store all together. Query metadata would be synchronized and annotated with additional metadata to a different data store, and the new API would interact with that to provide an API for querying and adding query metadata for each user. The downside was the added complexity and required effort to implement this, especially when compared to a solution that already exists and can be worked & improved on immediately. Additionally the synchronization process and ensuring some sort of consistency between UWS and the new store would require careful consideration and design, and a final issue would be that Firefly would have to interact with the new API using a non-VO standard way which is against their philosophy.

### 5.2 New FastAPI Query History App (Using UWS Database Storage)

A possible approach was to use the existing UWS database table, but introduce a different FastAPI web service for interacting with it. This would allow us to get around the limitations imposed by having to use the existing standard, as well leave open the door for implementing it as an aggregative service that collects queries from various UWS databases and and allows access to it & actions on the queries as if they were for one VO service. While there are potential benefits, the limitations are again the non VO-standard access which is problematic for Firefly, as well as the effort for creating a new RSP service.

## 6. Limitations and Concerns

Some thoughts and potential concerns that we may want to think about are listed here in this section:

### 6.1 {jobs} Endpoint Limitations

One limitation (perhaps can be argued against whether this is a limitation or not) is that the "jobs" endpoint only shows the following metadata: job id, ownerID, creationTime (optional)

Sample:

```xml
<uws:jobs xmlns:uws="http://www.ivoa.net/xml/UWS/v1.0" xmlns:xlink="http://www.w3.org/1999/xlink" version="1.1">
<uws:jobref id="ae5m9oso95k2t7wi">
<uws:phase>COMPLETED</uws:phase>
<uws:ownerId>user</uws:ownerId>
<uws:creationTime>2024-07-31T19:52:48.567Z</uws:creationTime>
</uws:jobref>
<uws:jobref id="afoh4if9romqx5yk">
<uws:phase>ERROR</uws:phase>
<uws:ownerId>user</uws:ownerId>
<uws:creationTime>2024-08-10T21:05:56.928Z</uws:creationTime>
</uws:jobref>
</uws:jobs>
```


This means that getting the actual query requires an extra jump to async/jobid. Specifically for a client like Firefly it's very likely that a field which may be useful to display in the list to users is the actual query string, which is not allowed by the standard in this list. It is always possible that a client will only show this additional metadata when a user triggers it via an additional action, but it's worth mentioning as it is a potential limitation.

### 6.2 Asynchronous vs. Synchronous Queries 

While asynchronous queries appear in the async job list, synchronous queries do not (Specifically in the CADC TAP service). In the CADC TAP service, sync queries do actually utilize the UWS database for storing and tracking information on the sync job, but this happens "behind the scenes", and they never actually create results in the results store, instead they stream the output directly and immediately. The job is tracked using the Post-redirect-Get (PrG) pattern.

Thus by design they do not want sync queries to appear in the list, and for this to be the case would have to be a configurable setting added specifically for Rubin.

Worth noting however that the standard also leaves open the interpretation that sync jobs should appear in the list if the job metadata is visible to the client so we would not be asking for something completely divergent to the standard.

In any case even if it proves difficult to convince CADC of the need for making this a configurable option, it should still be possible for us to add custom implementations of the relevant classes to enable this behavior in the CADC TAP Branch that we operate off of.

### 6.3 Scalability Considerations 

#### Growth projections 

Assuming no garbage collection, and considering that TAP is at the center of various data access tools (Firefly, Nublado, API & other clients) the number of incoming queries will likely grow the size of the UWS tables to a degree to which it's worth considering whether mitigation scenarios need to be designed to address scalability concerns.

Some stats & back-of-envelope calculations for how UWS table may grow (see Appendix 1).

#### Mitigation strategies

As it currently stands the CADC UWS implementation sets a default destruction time (1 week), after which the job no longer appears in the list, though it is still persisted in the UWS database. It's thus worth considering if we want this behavior to change so that the actual UWS job rows are deleted after the destruction time.

Similarly, HTTP DELETE actions don't seem to actually delete the job from the UWS database, which may or may not be the intended behavior if the database growth is a concern.

Relevant to this is always what the user actually expects from the system. Are they ok with their job completely disappearing after 1 week? Do they want by default all queries they have ever run to appear in their list? Or do they actually just want their "recent" queries, and perhaps if they do want to persist they can "Save" the query in the system?

### 6.4 Performance Concerns 

There are potential performance issues with use of UWS and /jobs endpoint worth considering as well:

For Firefly to show list of queries and include query text for each:

```
HTTP GET https://data.lsst.cloud/api/tap/async
   parse XML to UWS job list
   for each UWS job
       HTTP GET https://data.lsst.cloud/api/tap/async/jobid
           parse XML UWS job and add query to list

   return list
```

Imagine a user has 10 queries in their history to display, this leads to 11 HTTP Requests and XML parse actions.

A very rough estimate of how long this would take:

```
P = Time to parse single XML file ~= 20ms
H = HTTP Get request time ~= 300ms
I = Time to get initial list ~= 300ms
T = (P + H) * 10 + I ~= 3.5 seconds
```

(Missing additional possible delays due to networking latency, server load etc.) 

If the list of queries is 50 instead:

```
T ~= 16 seconds
```

#### Mitigation strategies 

- Limit how many queries a user can see. (Showing only 10 max sets an upper bound to response time)
- We could consider caching most X recent queries per user
- Don't show additional query info until requested (Only show ID & status)

### 6.5 Multi-Service Query History 

The async {jobs} endpoint is specific to the TAP Service endpoint it is available under.
Do users expect to see their queries for both TAP & SSOTAP at the same place? If so, that would require Firefly to combine results from /api/tap/async and /api/ssotap/async or SQuare to provide a new endpoint which gives them a combined view. This may extend to more than the two TAP services listed, which may end up introducing additional latency to the process of collecting user queries, unless they are collected and displayed individually (for example as different tabs).

## 7. Query Saving Functionality

Can a user save / archive a query with UWS as the query history store?
One of the long-term goals that has been included in the user query-history functionality is the ability to save a query (indefinitely). While it is potentially not part of an MVP in the context of the functionality described in the tech note, it's worth considering whether it is possible to be done if using UWS and the jobs endpoint as the source of user queries, or if it would require a separate query metadata store. A possible approach worth investigating is whether we can change the destructionTime to be set to 0 (or whichever the appropriate value is to annotate "indefinitely") when a user triggers the save action.

## Appendix 1: UWS Database Growth Calculations

Following is a very rough estimate of potential growth of the UWS database overtime.
The calculations assume 1000 active users.

Calculations on UWS database growth based on sample of a 11 day usage sample on data.lsst.cloud:

- Start Date: August 15th 2024
- End Date: August 26th 2024

Total Days: 11

Total Queries:
```sql
SELECT count(*) FROM uws.job
```
Result: 113,117

Number of users: 30

UWS has 4 tables, the relevant tables are job & jobdetail, the others are empty or have just 1 row.
The "job" table stores one row per query, the jobdetail stores several rows per query with various metadata.
	
Total size of uws.job (11 days):
```sql
SELECT pg_total_relation_size('uws.job') AS total_size;
```
Result: 36503552 ~= 35 Mb

Total size of uws.jobdetail (11 days):
```sql
SELECT pg_total_relation_size('uws.jobdetail') AS total_size;
```
Result: ~= 119.4 Mb

Average size per row (uws.job):
```sql
SELECT pg_total_relation_size('uws.job')::numeric / COUNT(*) AS avg_row_size FROM uws.job;
```
Result: ~322.35 bytes

Expected Database size for 1000 Users:
(Calculation was made by dividing the total size to ½ to exclude mobu queries and calculate an expected growth rate per user, then multiply that number by expected users = 1000).
Assuming half of these queries are mobu is a guess based on a quick analysis of the ownerID's of the users with the most queries in the system. Unfortunately in the version deployed as of writing the note, the ID does not show the username, so it is not possible to confirm.
In any case more precise calculations can be made if needed.

uws.job:
- After 1 year: 18.35 GB
- After 5 years: 91.75 GB

uws.jobdetail:
- After 1 year: 65.3 GB
- After 5 years: 326.5 GB

Non-indexed table scan duration for job.detail table after 5 years:
(Assuming SSD 200Mb/s read-speed)
334,336Mb/200Mb/s ~= 1671 seconds ~= 28 minutes

Non-indexed table scan duration for job table after 5 years: ~= 7 minutes

What columns are indexed currently:
- uws.job:
  - job_creationtime
  - job_ownerid
  - job_pkey

- uws.jobdetail:
  - jobdetail_fkey

Adding indexes to additional columns (phase for example) could be possible but will increase the size of the database (A few gigs per column would be a rough estimate for the above calculation and expected growth rate of number of queries).

If these are the only columns a query history client would filter on we can expect a few ms - seconds duration in the 5 year scenario.

