- [Automated Scheduling Engine | ASE](#automated-scheduling-engine---ase)
- [Jobs](#jobs)
  * [Lifecycle](#lifecycle)
    + [Sequence](#sequence)
  * [Components](#components)
    + [Path](#path)
    + [Full Path](#full-path)
    + [URI Arguments/Query Parameters](#uri-arguments-query-parameters)
    + [Headers](#headers)
    + [HTTP Method](#http-method)
    + [Body](#body)
    + [Constraints](#constraints)
  * [Constraints](#constraints-1)
    + [Time Constraint](#time-constraint)
    + [Pause Constraint](#pause-constraint)
    + [Approval Constraint](#approval-constraint)
    + [ServiceNow Constraint](#servicenow-constraint)
- [Settings](#settings)
  * [Process Lock](#process-lock)
  * [Review Mode](#review-mode)
  * [Admins](#admins)
- [API Reference](#api-reference)
- [Installation](#installation)
- [Technology](#technology)
  
# Automated Scheduling Engine | ASE
Part of HealthPartners IaaS Platform, ASE is responsible for scheduling micro function requests and handling micro function restrictions.  
  
[Full change log](https://gitlab.healthpartners.com/Emtech/ase/blob/master/CHANGELOG.md)
# Jobs
ASE creates instances of a `Job` when a proper request is received.  
  
## Lifecycle
Job's are created in ASE upon a properly formed request to the API (see [API Reference](#api-reference)), which is then processed for constraints and eventually resubmitted to the API Gateway.  ASE intercepts API Gateway traffic to prevent Jobs from triggering before they are ready.  A Job is held in ASE until it is free from all active Constraints or forcibly removed.  
### Sequence  
1. Incoming request is processed as a Job object and is stored in the Couchbase. All valid components of an incoming request are added to the Job (see [components](#components)). The jobQueue is fetched from Couchbase.
2. All jobs with `state` pending are checked for constraints associated with a Job  at each spin cycle (configured at 30 seconds) to validate if they are still active or not.  Job's with active constraints remain the same. Jobs with inactive constraints are made free of constraints associated to the job in Couchbase.
3. If no active constraints remain on a Job instance, the Job is resubmitted to the originating gateway with an SSH key signature on the request (for the gateway to validate the job finished processing and is allowed to run)
  
## Components
Each `Job` is comprised of several *components*:  
### Path
*Request path stripped of query params/uri args.*  
  
`Job.path`
```
/carpet/
```

### URI Arguments/Query Parameters
*URI arguments formatted as key:value dictionary*  
  
`Job.uriArgs`  
```
{
    "state": "running",
    "action": "promote"
}
```
### Headers
*Dictionary of headers used to create Job*  
  
`Job.headers`  
```
{
    "Authorization": "Basic c2FtcGxlbmFtZTpzYW1wbGVwdw==",
    "APIG-JID": "carpet-09122019-130130-9876",
    "Content-Type": "application/json",
    "host": "kong.healthpartners.com"
}
```
### HTTP Method
*HTTP method to process job to downstream service*  
  
`Job.method`  
```
POST
```
### Body
*Everything contained in HTTP request body, if applicable*  
  
`Job.body`  
```
{
    "inputs": {
        "os": "w2016r2",
        "prefix": "c",
        "app": "apig"
    }
}
```
### Constraints
*Active and inactive constraints associated with a single Job*  
  
See **Constraints** for more information  

`Job.constraints`  
```
[
    {
        "type": "time",
        "ACTIVE": True,
        "attributes": [
            { "notbefore": 123456789 }
        ]
    },
    {
        "type": "pause",
        "ACTIVE": True,
        "attributes": [
            {}
        ]
    }
]
```

## Constraints
Constraints are a property of Job that prevent it from exiting the job queue.  When a constraint is initialized, it is set with the property  
```
{ "active": True }
```  
A constraint is removed from a Job if a function flags `{ "active": False }` or just removes the Constraint object from the Job altogether.  
  
Every constraint has two properties: `type` and `attributes`.  
  
`type` declares the nature of the constraint and determines how it will be validated in the spin sequence.  
  
`attributes` are specific parameters unique to the constraint `type` and are used to validate a constraint during processing.  They may not be required depending on the constraint `type`.  
  
**More than one constraint of a certain `type` is allowed**  

### Time Constraint  
Provide upper and/or lower limits to *when* a Job can be submitted.  
`'type': 'time'`  
  
| Attribute Name | Description |
| --- | --- |  
| `notbefore` | epoch time referencing the earliest a Job can be submitted |  
| `notafter` | epoch time referencing the latest a Job can be submitted |  
| `nothours` | list of hours the Job cannot be submitted on (see example) |  
| `notdays` | list of days the Job cannot be submitted on (see example) |  
  
ASE sets `{ 'active': False }` once the current epoch time and hour or day compliments the `attributes` passed for the constraint.  

Currently jobs will be suspended indefinitely in the job queue if an improper combination of the attributes is submitted.  
  
Use the following conventions for `notdays`:  
  
| Day to Restrict | Allowable Values |  
| --- | --- |  
| Monday | `0`<br>`1`<br>`"MON"`<br>`"M"` |  
| Tuesday | `2`<br>`"TUE"`<br>`"T"` |  
| Wednesday | `3`<br>`"WED"`<br>`"W"` |  
| Thursday | `4`<br>`"THU"`<br>`"R"` |  
| Friday | `5`<br>`"FRI"`<br>`"F"` |  
| Saturday | `6`<br>`"SAT"`<br>`"S"` |  
| Sunday | `7`<br>`"SUN"`<br>`"U"` |  
  
For `nothours`, use **Military Time** and include every hour you wish to restrict.  Uses the clock in the Docker container running ASE.  
*To restrict any hour after 6PM and before 7AM...*  
```json
nothours: [18, 19, 20, 21, 22, 23, 24, 0, 1, 2, 3, 4, 5, 6]  
```  

#### Examples

**Example: Add New Job To Queue With A Time Constraint**
  
Request Body 
```
POST /

Authorization: Basic c2FtcGxlbmFtZTpzYW1wbGVwdw==
apig-jid: carpet-09122019-130130-9876
Content-Type: application/json  
  
{
    "inputs": {
        "name": "emtech",
        "newvm_shortname": "carpet01"
    },
    "constraints": [
        {
            "type": "time",
            "attributes": {
                "notafter": 12345678987,
                "notbefore": 234569876541
            }
        }
    ]
}
```  
  
**Example: Add New Job To Queue That Can't Run On Mondays, or Tuesdays Or Between 6PM and 7AM**
  
Request Body 
```json
POST /

Authorization: Basic c2FtcGxlbmFtZTpzYW1wbGVwdw==
apig-jid: carpet-09122019-130130-9876
Content-Type: application/json  
  
{
    "inputs": {
        "name": "emtech",
        "newvm_shortname": "carpet01"
    },
    "constraints": [
        {
            "type": "time",
            "attributes": {
                "notdays": ["MON", "TUE"],
                "nothours": [18,19,20,21,22,23,24,0,1,2,3,4,5,6]
            }
        }
    ]
}
```  
### Approval Constraint 
Stops a Job from processing until a number of Owners approve it or terminate it.  
`'type': 'approval'`  
  
| Attribute Name | Description |
| --- | --- |  
| `min_approval_count` | Number of owners that must approve the Job |  
  
ASE sets `{ 'active': False }` once the number of unique Owners send approvals to the Job that exceeds the `min_approval_count` configured at the micro function layer.    

*Approval Constraints are configured at the micro function level and cannot be manually added.  

### Pause Constraint 
Indefinitely suspends the Job in the queue.    
`'type': 'pause'`  
  
| Attribute Name | Description |
| --- | --- |  
| | |  
`pause` constraint does not have any attributes  
  
ASE sets `{ 'active': False }` only after an API call is made to explicitly remove the pause constraint on a job.
 
***Example: Add New Job To Queue With Pause Constraint***
  
Request Body 
```
POST /

Authorization: Basic c2FtcGxlbmFtZTpzYW1wbGVwdw==
apig-jid: carpet-09122019-130130-9876
Content-Type: application/json  
  
{
    "inputs": {
        "name": "emtech",
        "newvm_shortname": "carpet01"
    },
    "constraints": [
        {
            "type": "pause",
            "attributes": {}
        }
    ]
}
```  
Response Body  
```
HTTP 200 OK
Content-Type: application/json  

{}
```  
  
### ServiceNow Constraint 
Suspends the job in the queue until specific conditions of an object in ServiceNow are met.  

This constraint watches TASKs in ServiceNow for specific states.  When all TASKs match the state provided, the job is allowed through.  
  
Uses a ServiceNow Scripted API object to interact - this object is managed as a partnership between ServiceNow & Emtech.  Read more at:  
https://gitlab.healthpartners.com/server-retirement/servicenow-task-creation  
  
User provides a list of `target_states` that every task must meet, and a list of `terminate_states` that force current Job into a `terminated` status.  
  
Additional Behavior:  
- If a TASK does not exist with that short description, Job will wait.  
- If ***any*** TASK has a state listed in `terminate_states`, Job is immediately `terminated`  
- Parent record can be any valid ServiceNow object that has children tasks or change tasks.  The API checks the letters in the record to determine which tasks to look at.
- If `parent_record` does not exist, Job is immediately `terminated`   
- If Job is `terminated`, an event is added to the timeline explaining why
  
| Attribute Name | Description | Example |
| --- | --- | --- |  
| `parent_record` | Any valid RITM or CHG ticket number | `RITM012345` |  
| `task_short_descriptions` | The short_descriptions of the TASKs that are children records of the `parent_record`.<br><br>Case insensitive. | `Install Commvault`<br>`shut down server` |  
| `target_states` | List of allowable states for every TASK to continue release constraint<br><br>Defaults to<br>[`closed_complete`, `closed`] | [`work_in_progress`, `closed_complete`] |  
| `terminate_states` | List of states that immediately trigger Job to `terminate`<br><br>Defaults to<br>[`closed_cancelled`, `closed_skipped`, `cancelled` ] | [`closed_cancelled`] |
  
  
***Example: Wait For Two Tasks To Be In Progress Or Finished - Terminate On Cancelled or Skipped***
  
Request Body 
```json
  
{
    "inputs": {
        "name": "emtech",
        "newvm_shortname": "carpet01"
    },
    "constraints": [
        {
            "type": "servicenow",
            "attributes": {
                "parent_record": "RITM456789",
                "task_short_descriptions": [
                    "Confirm Retirement",
                    "Shut down server"
                ],
                "target_states": [
                    "work_in_progress",
                    "closed_complete"
                ],
                "terminate_states": [
                    "closed_cancelled",
                    "closed_skipped"
                ]
            }
        }
    ]
}
```  

# Settings
ASE has various settings that affect how jobs are processed.

## Process Lock
The *process lock* is a setting that allows or disallows jobs from exiting ASE.  
- If the process lock is unlocked, Job Lifecycle will continue as normal.
- If the process lock is locked, no Jobs will be submitted back to APIG unless explicitly `submitted` [through the API](https://gitlab.healthpartners.com/Emtech/ase/blob/master/DOC/API-LATEST.md).  
  
Only Admins can update the process lock.

## Review Mode
*Review Mode* is a list of micro functions that will automatically be given a Pause Constraint when created in ASE.  The purpose is to provide a processing filter on ASE without using the Process Lock on micro functions that require special attention.  
  
For instance, if the current Review Mode list includes `hello-world`, and a new job is submitted with the JID `hello-world-20191212-112105-6731`, it will be given a Pause constraint and remain in ASE until the contraint is removed or the Job is forcefully submitted.

## Admins
Admins in ASE are a static list controlled by IS&T Emtech.

# API Reference

[**API Version 1 Reference**](https://gitlab.healthpartners.com/Emtech/ase/blob/master/DOC/API-LATEST.md)
[**Legacy API - Unsupported**](https://gitlab.healthpartners.com/Emtech/ase/blob/master/DOC/API-LEGACY.md)

# Installation

```
cd ase
yum -y install docker
systemctl enable --now docker
docker build -t ase .
docker volume create asedata
docker run -p 5001:5001 --name ase --volume asedata:/opt/ase/data  --network backend -e "ASE_DATA_DIR=/opt/ase/data"  -e "APIG_SERVER=kong.healthpartners.com" -e "SN_CREDENTIALS=<Sn creds Base64 encoded>" -d ase
```

# Technology
ASE is a Python 3 application designed to run in a Docker container
