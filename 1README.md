# Conductor 
- [Summary](#summary)
    + [Requirements:](#requirements-)
- [Playbook Inputs](#playbook-inputs)
- [Workflow](#workflow)
  * [Job Plan Initialization](#job-plan-initialization)
  * [Job Completion Loop](#job-completion-loop)
      - [Check running job completion states](#check-running-job-completion-states)
      - [Expose children job outputs](#expose-children-job-outputs)
      - [Ready-up waiting jobs](#ready-up-waiting-jobs)
      - [Start jobs that are ready](#start-jobs-that-are-ready)
- [Other Variables](#other-variables)
  * [conductor](#conductor)
  * [children_jobs](#children-jobs)
      - [jid](#jid)
      - [begin_wait_time](#begin-wait-time)
      - [all_jids](#all-jids)
      - [status](#status)
- [Omitting or Including Micro Functions](#omitting-or-including-micro-functions)
  * [if_any_inputs](#if-any-inputs)
      - [equal](#equal)
      - [exist_for](#exist-for)
      - [do_not_equal](#do-not-equal)
  * [only_when_inputs](#only-when-inputs)
      - [equal](#equal-1)
      - [exist_for](#exist-for-1)
      - [do_not_equal](#do-not-equal-1)
  * [do_not_track](#do-not-track)
- [buildParams](#buildParams)
  * [Example](#example)
- [Outputs](#outputs)
  * [Micro Function Outputs](#micro-function-outputs)
      - [Example](#example)
  * [Dynamic Inputs](#dynamic-inputs)
- [Recovery Job](#recovery-job)
- [Resubmitting Incomplete Jobs](#resubmitting-incomplete-jobs)
  * [Example](#example-1)

# Documentation
[Full Release Log](https://gitlab.healthpartners.com/Emtech/conductor/blob/master/DOCS/RELEASE.md)

# Summary  
### Requirements:
- Python 3
- urllib3 (Python module)


This playbook is designed to receive it's inputs from the `host_vars` against which it is being executed and can thereby execute any `conductors` made available (described in detail below).  

Conductor requires one var, `conductor`, which represents a set of micro functions arranged as a dictionary:  

```yaml
conductor:

  # conductor one = "minuet"  
  minuet:
    microfunction_one:
      ...
    microfunction_two:
      ...
    microfunction_three:
      ...

  # conductor two = "get_and_register_new_hostname"
  get_and_register_new_hostname:
    microfunction_one:
      ...
    microfunction_two:
      ...
    microfunction_three:
      ...

```

`conduct.yml` will parse through a dictionary, `conductors`, and read the chosen conductor dictionary as `{{ conductors[conductor] }}`.

*In the example above, passing* `conductor: minuet` *would cause this playbook to orchestrate the micro functions listed in the* `minuet` *dictionary*.  

# Playbook Inputs  
The following are required `host_vars` or `extra_vars`:  
- `apig_host` *API Gateway shortname... "kong-sandbox"*
- `conductor`  *Sub-dictionary of "conductors" in this playbook to call... "minuet"*

The following are optional `host_vars` or `extra_vars`:  
- `apig_jid` *JID assigned to job.  Used for resubmitting jobs... "minuet-121009-080045-1234"*
- `infra_jid` *A previous JID provided to build the apig_jid conductor microfunction inputs.  Used for dependency dynamic inputs ... *

`conductor` dictionaries (inside `conductors`) are a named collection of micro functions with a set of required and optional parameters.  See **[Other Variables](#other-variables)**.  


# Workflow
The playbook operates in two stages:  
- **Job Plan Initialization**  
- **Job Completion Loop**
  - *Check running job completion states*
  - *Ready-up waiting jobs*
  - *Start jobs that are ready*

## Job Plan Initialization
Script checks if the `apig_jid` already has a `children_jobs` entry in API-G storage and creates one if it doesn't exist.  

The variable `children_jobs` is proprietary to the process (created and referenced by the script) and represents the blueprint for how the full collection of micro functions in the chosen `conductor` will be called and has the following structure:  
```json
{
  "children_jobs": {
    "mf-one": {
      "status": "complete",
      "jid": "get-new-hostname-20190927-1232-1036",
      "all_jids": {
          "get-new-hostname-20190927-1232-1036": "successful",
          "get-new-hostname-20190927-1200-1921": "failed"
      }
    },
    "mf-two": {
      ...
    }
  }
}
```  
For a new `apig_jid`, the process will intake the `conductor` dictionary and build `children_jobs` to expose each micro function invocation in an appropriate order based on its dependencies (if it has any) and/or restrictions (see [Omitting or Including Micro Functions](#omitting-or-including-micro-functions)).  The process updates the `status` and populates the `jid` and `all_jids` sections as execution proceeds.  All jobs are marked as `waiting` at execution time for a new `apig_jid`.  Re-submission of an `apig_jid` handles `children_jobs` differently (see [Resubmitting Incomplete Jobs](#resubmitting-incomplete-jobs))

## Job Completion Loop  
The `conductor` is complete when all jobs in `children_jobs` have `"status": "complete"` (***success condition***) or there are jobs with `"status": "waiting"` that have failed dependencies (***fail condition***).  

The loop cycles through three stages:  
- *Check running job completion states*
- *Expose children job outputs*
- *Ready-up waiting jobs*
- *Start jobs that are ready*

#### Check running job completion states  
For every child micro function in `children_jobs` with `"status": "running"`, check the `status` field in API-G storage to validate if job was successful or not.  

- **successful** - Job will be marked in `children_jobs` as `"status": "complete"`
- **failed** - Job will be marked in `children_jobs` as `"status": "failed"`
- **failed (with `"continue_on_fail": true` in MF definition)** - Job will be marked in `children_jobs` as `"status": "complete"`  

Jobs will be retried up to five times if no value is provided for `max_retries_on_failure` for the micro function such as:  

```yaml
microfunction_name:
  ...
  max_retries_on_failure: 10
  ...
```  

A single job will timeout after the run-time exceeds the value set for `timeout_in_minutes` parameter.  The default `timeout_in_minutes` is currently `600` which equates to *10 hours*.

#### Expose children job outputs
For every child micro function in `children_jobs` with `"status": "complete"` and a corresponding entry in the `conductor` var for `outputs` the process will parse the value from the `jid` listed in `children_jobs` and add it to a variable `build_parameters` which is eventually posted to the `apig_jid` in API-G storage:  
- Check micro function is complete,  
- Read `outputs.var_loc` from the `conductor` definition,
- Parse from `jid`'s API-G storage entry,
- Place value in `build_parameter` script variable

#### Ready-up waiting jobs
For every child micro function in `children_jobs` with `"status": "waiting"`, check if any dependencies are completed.  
- dependencies are listed in `conductor[microfunction]["depends_on"]`
- the statuses *for the dependent jobs* listed in `children_jobs` are reviewed for completion (`"status": "complete"`)  

Satisfying all criteria will result in a change to `"status": "ready"` in `children_jobs`  

#### Start jobs that are ready
For every child micro function in `children_jobs` with `"status": "ready"`, call the micro function through the API Gateway (`apig_host`).  

Inputs listed in `conductor` are either **static** (non-variable, no dependencies from upstream jobs) and **dynamic** (originate as outputs from upstream jobs).  See [Dynamic Inputs](#dynamic-inputs) for more information about how dynamic inputs are parsed.  

For example, the declaration in `conductor` for a micro function that leaves the comment "Please contact HealthPartners IS&T if you have any concerns" (*static* input) against a computer whose name depends on the output of an upstream job (*dynamic* input) might be:  
```yaml
conductors:
  example_conductor:

    example_microfunction:

      endpoint: "micro-function"
      inputs:
        comment: "Please contact HealthPartners IS&T if you have any concerns"
        computer_name: "%%computerName"
      ...
      ...
```

See [Outputs](#outputs) for more details.  

The micro function is called at `apig_host` for the `endpoint` with a set of credentials and inputs listed in the `conductor` definition.  

For the example above, assuming that `buildParams` included `computerName = my-laptop`, the micro function would be triggered with:  

`POST` https://<apig-host>.healthpartners.com/micro-function/  
```json
{
  "inputs": {
    "comment": "Please contact HealthPartners IS&T if you have any concerns",
    "computer_name": "my-laptop"
  }
}
```


# Other Variables  
## conductor  
`conductor` is usually a sub dictionary of `conductors` defined in group_vars of this playbook and represent a series of micro functions for orchestration.  

In the example below, `example_conductor` is a sub conductor of `conductors` that can be called by starting this playbook with the var:  
```yaml
...
conductor: "example_conductor"
...
```
```yaml
conductors:

  example_conductor:

    example_microfunction:
      inputs:
        # DICT of explicit key:pair values to send in inputs payload (OPTIONAL) (static)
        # DICT of keys paired with a variable name to substitute for the value.  Any given key:pair will be omitted from outgoing payload if the variable name does not resolve at time of invocation (OPTIONAL) (dynamic)
      outputs: # LIST of dicts
      - var_name: # name of var to assign to the parent variable, "build_parameter"
        var_loc: # location to read the output variable from the job info in API-G storage for jid
      auth_user: # username to authenticate to downstream API-G service (recommended to be obfuscated with Ansible Vault)
      auth_password: # password to authenticate to downstream API-G service (recommended to be obfuscated with Ansible Vault)
      omit_when_false: # DICTIONARY of criteria to exclude this entry from the job plan (children_jobs) if found to be false (OPTIONAL)
        example_var: "example_val" # micro function will only be included if "example_var" is equal to "example_val"
      continue_on_fail: # boolean used to declare the conductor can continue even if this micro function fails (OPTIONAL)
      depends_on: # LIST of micro functions that must be complete prior to execution of this micro function
      max_retries_on_failure: # INTEGER used to determine how many times a micro function will be resubmitted if it fails.  Default = 0 (OPTIONAL)
      timeout_in_minutes: # INTEGER used to determine how long a microfunction will be checked after it is set to RUNNING.  Default = 600 (OPTIONAL)
      do_not_track: true # boolean used to declare the microfunction  need not be tracked in the children_jobs plan which does mean that the conductor need not wait for the comppletion of this microfunction.
      include_me: # DICTIONARY used to declare conditions to include this micro function.  All microfunctions are included by default unless restricted herein (OPTIONAL)
        if_any_inputs: # DICTIONARY (OPTIONAL)
          equal: # LIST of key: pair values.  At least one of these must explicitly match the initial inputs of the conductor else this micro function will be omitted. (OPTIONAL)
          exist_for: # LIST of keys.  At least one of these must explicitly match a key in the inputs of the conductor else this micro function will be omitted. (OPTIONAL)
          do_not_equal: # LIST of key: pair values.  If any one of these explicitly matches the initial inputs of the conductor this micro function will be omitted. (OPTIONAL)
        only_when_inputs: # DICTIONARY (OPTIONAL)
          equal: # LIST of key: pair values.  All of these must explicitly match the initial inputs of the conductor else this micro function will be omitted. (OPTIONAL)
          exist_for: # LIST of keys.  All of these must explicitly match a key in the inputs of the conductor else this micro function will be omitted. (OPTIONAL)
          do_not_equal: # LIST of key: pair values.  If any one of these explicitly matches the initial inputs of the conductor this micro function will be omitted. (OPTIONAL)
```  

See [Omitting or Including Micro Functions](#omitting-or-including-micro-functions) for information about using `include_me`.

## children_jobs
`children_jobs` is used to execute the conductor in a specific order.

```json
{
  "children_jobs": {
    "mf-one": {
        "jid": "get-new-hostname-20190927-1232-1036",
        "begin_wait_time": 1571256547,
        "all_jids": {
            "get-new-hostname-20190927-1232-1036": "successful",
            "get-new-hostname-20190927-1200-1921": "failed"
        },
        "status": "successful"
    },
    "mf-two": {
        "jid": "create-host-record-20190927-1313-1292",
        "begin_wait_time": 1571256517,
        "all_jids": {
            "create-host-record-20190927-1313-1292": "failed"
        },
        "status": "failed"
    },
    "mf-linux": {
        "jid": "",
        "begin_wait_time": 1571256537,
        "all_jids": {},
        "status": "waiting"
    },
    "mf-windows": {
        "jid": "",
        "begin_wait_time": 1571256527,
        "all_jids": {},
        "status": "waiting"
    }
  }
}
```  

#### jid
The currently invoked job the conductor is checking on  
#### begin_wait_time
The start time of the current `jid`
#### all_jids
Dictionary of all jids used in the micro function call and their status'
#### status
Status of the current micro function.  Potential values include `waiting`, `running`, `failed`, `successful`, or `timeout`

# Omitting or Including Micro Functions
Special configuration options exist for building dynamic micro function orchestrations:
```yaml
conductors:
  conductor_name:
    micro_function_x:
      ...
      ...
      include_me:
        if_any_inputs:
          equal:
            - foo: bar
            - company: optum
          exist_for:
            - baz
            - qux
            - zap
          do_not_equal:
            - foo: goo
        only_when_inputs:
          equal:
            - os: "windows"
            - prefix: "e"
            - site: "1180"
          exist_for:
            - request_number
            - request_name
          do_not_equal:
            - os: "linux"
```  

The above layout is ***particularly restrictive*** for demonstrative purposes.  All values in `include_me`, including `include_me`, are **optional**.  

    All values are reviewed against the initiating job inputs
    (queried out of API-G storage) for exclusion criteria.  

## if_any_inputs
This section is reviewed first and given lowest precedence, i.e. values from `only_when_inputs` take precedence in case of conflict.  

#### equal
At least one key:pair in this list must equal an entry in the job inputs otherwise the micro function will be omitted from the job.  
**Example Inputs**  
```json
{
  "inputs": {
    "os": "windows",
    "prefix": "e"
  }
}
```  

**Micro function that will be included...**  
```yaml
include_me:
  if_any_inputs:
    equal:
      - os: "windows"
```  

**Micro function that will be omitted...**  
*Because `os` does not equal `rhel` in inputs*  
```yaml
include_me:
  if_any_inputs:
    equal:
      - os: "rhel"
      - prefix: "e"
```

#### exist_for
At least one value in this list must be provided as a key in inputs else the micro function will be omitted from job.
**Example Inputs**  
```json
{
  "inputs": {
    "sn_req_ritm": "RITM0123456",
    "os": "windows"
  }
}
```  

**Micro function that will be included...**  
```yaml
include_me:
  if_any_inputs:
    exist_for:
      - ritm
```  

**Micro function that will be omitted...**  
*Because no entry for `prefix` is provided by inputs*
```yaml
include_me:
  if_any_inputs:
    exist_for:
      - prefix
```  

#### do_not_equal
If at least one key:value in this list exists in inputs the micro function will be omitted from job.  Used to explicitly block micro functions from being included in a plan if a specific key:value pair is supplied as an `input`
**Example Inputs**  
```json
{
  "inputs": {
    "os": "windows"
  }
}
```  

**Micro function that will be included...**  
```yaml
include_me:
  if_any_inputs:
    do_not_equal:
      - os: rhel
```  

**Micro function that will be omitted...**  
```yaml
include_me:
  if_any_inputs:
    do_not_equal:
      - os: windows
```  

## only_when_inputs
This section is reviewed last and given top precedence, i.e. values from `only_when_inputs` take precedence in case of conflict.  

#### equal
All key:pair values in this list must have an equivalent entry in the job inputs otherwise the micro function will be omitted from the job.  
**Example Inputs**  
```json
{
  "inputs": {
    "foo": "bar",
    "os": "windows"
  }
}
```  

**Micro function that will be included...**  
```yaml
include_me:
  if_any_inputs:
    equal:
      - foo: "bar"
      - os: "windows"
```  

**Micro function that will be omitted...**  
*Because no `os` value is included in inputs*
```yaml
include_me:
  if_any_inputs:
    equal:
      - foo: "bar"
```

#### exist_for
All values in this list must be provided as a key in inputs else the micro function will be omitted from job.
**Example Inputs**  
```json
{
  "inputs": {
    "sn_req_ritm": "RITM0123456",
    "foo": "bar",
    "os": "windows"
  }
}
```  

**Micro function that will be included...**  
```yaml
include_me:
  if_any_inputs:
    exist_for:
      - ritm
      - os
```  

**Micro function that will be omitted...**  
*The inputs is missing the `prefix` key*
```yaml
include_me:
  if_any_inputs:
    exist_for:
      - ritm
      - os
      - prefix
```  

#### do_not_equal
If at least one key:value in this list exists in inputs the micro function will be omitted from job.  Used to explicitly block micro functions from being included in a plan if a specific key:value pair is supplied as an `input`
**Example Inputs**  
```json
{
  "inputs": {
    "os": "windows"
  }
}
```  

**Micro function that will be included...**  
```yaml
include_me:
  if_any_inputs:
    do_not_equal:
      - os: rhel
```  

**Micro function that will be omitted...**  
*Because `os` does not equal `windows`*
```yaml
include_me:
  if_any_inputs:
    do_not_equal:
      - os: windows
```  
## do_not_track
This is a boolean value and the microfunction with this tag `do_not_track` will not be tracked in the conductor plan.This means the parent job doesnt not worry about the child job. It doesnt even store the jid in the conductor plan.


**Micro function that will be omitted in the parent job plan...**  
```yaml
do_not_track: true
``` 
# buildParams
If an `infra_jid` is specified to the conductor as an input, all the `buildParams` value of the infra_jid would be copied to the  current conductor's `buildParams` value.

If an `infra_jid` is not specified to the conductor as an input, During execution of the conductor job, any micro functions with a defined `outputs` dictionary will be queried for defined values which are then added to the conductor's `buildParams` value.

## Example
Given a infra_jid with jid `asp-120012-071221-48` and the conductor will `getPublishedJobInfo()` for jid which returns
```json
{
  "jid": "asp-120012-071221-48",
  "start_time": "120012342342",
  "buildParams": {
      "os": "w2016",
      "host_name": "e01023",
      "site": "Mendota Heights",
      "production": true
  }
}
```  

And assuming `buildParams = {}` (because there were no inputs to initialize the job),

 
First the conductor will attempt to read the value at `buildParams` of infra_jid, then copy it to  `buildParams` of the current conductor resulting in  
**buildParams**  
```json
{
  "host_name": "e01023",
  "site": "Mendota Heights",
  "os": "w2016",
  "production": true
}
```  
If the conductor has inputs, the inputs would also be added to the `buildParams`


# Outputs
During execution of the job, any micro functions with a defined `outputs` dictionary will be queried for defined values which are then added to the conductor's `buildParams` value.  

## Micro Function Outputs  
A given micro function can define outputs as:  
```yaml
outputs:
  - var_name: hostname
    var_loc: registeredVars.localhost.host_name
  - var_name: datacenter
    var_loc: registeredVars.localhost.site
```

After a micro function is marked as successful and complete, conductor will read the published information for the `apig_jid` specific to the micro function, parse the data at the `var_loc`, and assign the parsed value to `var_name` in the conductors `buildParams` variable.  

#### Example
Given a micro function with jid `get-host-information-120012-071221-48` and the micro function definition:  
```yaml
get_host_information:
  ...
  ...
  outputs:
    - var_name: hostname
      var_loc: registeredVars.localhost.host_name
    - var_name: datacenter
      var_loc: registeredVars.localhost.site
```  

And assuming `buildParams = {}` (because there were no inputs to initialize the job),

Once `get_host_information` is marked as `COMPLETE` the conductor will `getPublishedJobInfo()` for jid `get-host-information-120012-071221-48`, which returns
```json
{
  "jid": "get-host-information-120012-071221-48",
  "start_time": "120012342342",
  "registeredVars": {
    "localhost": {
      "host_name": "e01023",
      "site": "Mendota Heights"
    }
  }
}
```  
First the conductor will attempt to read the value at `var_loc`, then assign it to `var_name` in `buildParams` resulting in  
**buildParams**  
```json
{
  "hostname": "e01023",
  "datacenter": "Mendota Heights"
}
```  

When a job is listed with dynamic_inputs as such:  
```yaml
micro_function_two:
  ...
  ...
  inputs:
    os: "windows"   # considered as static input
    newhostname: "%%hostname" # considered as dynamic input
    newsite: "%%datacenter"   # considered as dynamic input
  ...
  ...
```  

...calling the micro function after the previous example completed results in an outgoing payload of:  
```json
{
  "inputs": {
    "os": "windows",
    "newhostname": "e01023",
    "newsite": "Mendota Heights"
  }
}
```  
## Dynamic Inputs
**Dynamic inputs always read from `buildParams` and are omitted if the value provided does not resolve**  
**Dynamic Inputs are always preceeded with `%%` for the conductor to recognise it as `dynamic` input**
Dynamic inputs always read from `buildParams` which is potentially updated throughout the job. Dynamic Inputs may also contain some static text in the input value with a dynamic input precceded with `%%`. In configuration, the *value* listed in the input is used to parse `buildParams` so if a micro function is defined as:  
```yaml
micro_function_two:
  ...
  ...
  inputs:
    os: "windows"
    newhostname: "Hostname: %%hostname"
    newsite: "%%datacenter"
  ...
  ...
```  

...this is equivalent to:  
```yaml
micro_function_two:
  ...
  ...
  inputs:
    os: "windows"
    newhostname: "Hostname: {{ buildParams['hostname'] }}"
    newsite: "{{ buildParams['datacenter'] }}"
  ...
  ...
```  

# Recovery Job
**Recovery response can be included for any job for the series of action to take place if the child Job fails**  

Dynamic inputs  for the recovery job always read from `buildParams` which is potentially updated throughout the job. If  `include_failure_as_input` is set to True , the recovery job may have the failuresResults of the child job in the payload and is an optional flag for the job
```yaml
micro_function_two:
  ...
  ...
  
  endpoint: "hello-fraunchless"
    inputs:
      custom_failure: "mf-two"
      dynm_one: "%%os"
    outputs:
    - var_name: "mf_two_message"
      var_loc: "registeredVars.localhost.sup.msg"
    auth_user: "{{ generic_username }}"
    auth_password: "{{ generic_password }}"
    recovery_response:
      endpoint: "hello-fraunchless"
      include_failure_as_input: true
      inputs:
        microfunction: "hello world"
        os: "%%os"
      auth_user: "{{ generic_username }}"
      auth_password: "{{ generic_password }}"
  ...
  ...
```  


``` 
# Resubmitting Incomplete Jobs
To resubmit a job, simply make a `POST` request to the original endpoint used and include the `apig_jid` in the body (or `APIG-JID` header) with any additional or changed inputs required for the job.  The API Gateway will re-use any existing inputs used in previous runs and combine new inputs.  

For example, suppose an initial request is made as  

```json
{
  "inputs": {
    "os": "w2016",
    "site": "1180",
    "cpu": 4
  }
}
```  
And the process returns `apig_jid = conductor-12345`  

If the process failed and the `cpu` needs to change from **4** to **6**, the request could be resubmitted as:  

```json
{
  "inputs": {
    "apig_jid": "conductor-12345",
    "cpu": 6
  }
}
```  
...and a `cpu` of **6** will be used along with `os = w2016` and `site = 1180` (because they were used initially)


If `children_jobs` is found in parent `apig_jid` entry in API-G storage it will be used instead of generating a new `children_jobs` var.  Any jobs not set with `"status": "completed"` will be reset to `"status": "waiting"` and the job loop will begin with fresh JIDs for anything not previously `completed`.

## Example  
The conductor represents a collection of micro functions for orchestration by `conductor.py`:  

```yaml
---
conductor:

  generic:

    mf-one:
      endpoint: "hello-fraunchless"
      inputs:
        custom_failure: "mf-one" # considered as static input
        foo: "bar"               # considered as static input
        dynm_one: "%%os"         # considered as dynamic input
      outputs:
      - var_name: "mf_one_message"
        var_loc: "registeredVars.localhost.sup.msg"
      max_retries_on_failure: 3
      auth_user: "testy"
      auth_password: "testyPassword"
      continue_on_fail: true
      timeout_in_minutes: 1
      include_me:
        if_any_inputs:
          equal:
            - os: "w2016"
          exist_for:
            - ritm
          do_not_equal:
            - os: "w2019"
        only_when_inputs:
          equal:
            - os: "w2012r2"
          exist_for:
            - ritm
          do_not_equal:
            - os: "rhel7"

    mf-two:
      endpoint: "hello-fraunchless"
      inputs:
        custom_failure: "mf-two"   # considered as static input
        dynm_one: "%%os"           # considered as dynamic input
      outputs:
      - var_name: "mf_two_message"
        var_loc: "registeredVars.localhost.sup.msg"
      auth_user: "testy"
      auth_password: "testyPassword"
      recovery_response:
        endpoint: "hello-fraunchless"
        include_failure_as_input: true
        inputs:
          microfunction: "hello world"   # considered as static input
          os: "%%os"                     # considered as dynamic input
        auth_user: "{{ generic_username }}"
        auth_password: "{{ generic_password }}"

    mf-three:
      endpoint: "hello-fraunchless"
      inputs:
        custom_failure: "mf-three"    # considered as static input
        dynm_one: "Operating System : %%os"   # considered as dynamic input with some static text
        random_mess: "%%mf_one_message"       # considered as dynamic input
      outputs:
      - var_name: "mf_three_message"
        var_loc: "registeredVars.localhost.sup.msg"
      auth_user: "testy"
      auth_password: "testyPassword"
      do_not_track: true
```
