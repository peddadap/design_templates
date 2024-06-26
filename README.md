## Table of Contents
- [Preface](#preface)
- [Overview](#overview)
- [System Design](#system-design)
- [Flow Design](#flow-diagram)
- [Sequence Design](#sequence-flow)
- [Data Quality](#dq-framework)
    * [Data Quality Data Model](#dq-data-model)
    * [Data Quality Data Flow](#dq-data-flow)
    * [Recovery Data Flow](#recovery-data-flow)
- [Deployment Model](#deployment-model)

## Preface 

The design scope here covers areas that are either being reengineered or/and extended to enhance the current functionality of several systems at spike up, most noticeably airbyte,airflow and snowflake. A brand new system for operational management is being introduced, more on this below.  

There are 3 parts/modules to Phase-1 release. The High Level Design here will try and address the specifics of each of the modules

On the immediate need, Module 1, consists of the following sub-modules -- please see the project plan on details of Module 1 and its parts
 
- 1.1: `Stage-1: Raw Zone  Re-Engineered Pipelines = Casino Operator Source --> AWS S3-Files`
- 1.2: `Stage-2: Raw Zone S3 Files Black Box DQ Checks`
- 1.3 `RAW Data S3 Files to Snowflake Raw Zone Linkage`

## Overview

![alt text](./spike_overview.jpg)

#### Core Systems as part of overall platform

- AirByte  
    * Align connector development around low code framework like  Airbyte YAML CDK
    * A path to decommission current source connectors that have significant code foot print 
    * A second level of generalization around YAML connectors on common operations like custom authentication, specialized request construction 
      and wide format response parsing
    * Component registry build and deployment for reusability

- AirFlow
    * Re-engineered DAG's to improve system utilization by collating and parallelizing platform operations 
    * Stage - I Data Quality inspection and control flow of Airflow jobs
    * Better Diagnostics and Error reporting of flows on AirFlow

- Op Control Center

    * Decouple Business Op activity from airbyte connector framework 
    * Data Source Management --  new connections, repair or fix connectivity and maintenance of source data assets 
    * Dedicated OP control center with near realtime monitoring and activity logs

- S3 RAW Zone

    * All sourced datasets will be stored on AWS S3 as json files
    * Sources extracted will be partitioned by time and/or with other context appropriate for optimal downstream consumption

- SnowFlake

    * Externalize Raw Sources to cheaper alternative like S3
    * Phase-I, source datasets will appear as external tables inside SnowFlake
    * Phase II and beyond will engineer Snowflake for optimal compute and storage

## System Design
    
### AirByte Changes

- The components colored in green are areas where design and behavior of the existing system will be altered  
- The Airbyte YML CDK extensions are python modules that abstract logic based on needed behavior for a given platform
    * Custom Authenticator ; When authentication falls outside basic and simple token model
    * Custom Requester ; Requires packing pre-computed payloads that have to be dynamically formed
    * Custom Record Extractor ; When response parsing is warranted, for example converting xml to json payload
    * With the ability to enhance behavior as needed

- The Airbyte YML CDK extensions on Destination connectors is optional, may be needed to inject Data Quality Checks, will be refined as part DQ-Framework Design



```mermaid
flowchart LR
   A1(Source \n Rest API)
    subgraph AirByte-Jobs
        subgraph Connections-yML 
            subgraph Source
                A2(Source \n Connector)
                A2.1(Airbyte YML \n CDK extensions):::module
            end
            A3(Connection)
            subgraph Destination
                A4(Destination \n Connector)
                A4.1(Airbyte YML \n CDK extensions):::module
            end
            A5( DQ  \n Transient Storage):::module
        end
    end
    classDef module stroke:#0f0
    A1-->AirByte-Jobs
    Source-->A3
    A3-->Destination
    A4-->A5


```
### AirFlow Changes

- The components colored in green are areas where design and behavior of the existing system may be altered 
- The AWS s3 buckets will have a logical structure where each platform will sit under a hierarchal root  
- Under the Platform hierarchical root, each unique data-stream will be written by date and time as sourced
- The hierarchy goes somewhat like  `"Platform_Name > "Operator Account" > "Stream_Name" > "YYYY" > "MM" > "DD" > {yyyy}_{mm}_{dd}_{H24ssms}.json`,
- At the very top, each platform will have a designated bucket 
- User/ Admin will be able to view and  drill down the folders in the hierarchy
- The S3 is an object store, though AWS S3 console gives users the ability to drill/navigate the folders, these are not actual folder rather the Key itself for that given object

##### Visual Map of S3 
```mermaid
graph TD
    A(CellExpert)
    A -->B(Operator-1)
    C(Mexos) -->C1.1(Operator-1)
    B --> D(RegistrationReportStream)
    B --> E(DynamicVariablesReportStream)
    D --> I(2024/02/12/)
    I --> L[20240212_235900000.json]
    C1.1 --> F(Stream-1)
    C1.1 --> G(Stream-2)
    F --> H( ........)
    G --> J( ........)


```
##### Typical Airflow Flow Control

```mermaid
flowchart LR
   subgraph "AirFlow"
        G(Scheduler)
        subgraph "Typical-DAG"
            subgraph "AirByte-TiggerSync"
                1.1[Platform1-OA1]
                1.2[Platform1-OA2]
                1.3[Platform1-OA ..N]
            end
            subgraph "Transient-Location"
                2.1[Platform1-OA1]
                2.2[Platform1-OA2]
                2.3[Platform1-OA ..N]
            end
            
            subgraph "DQ"
                3.1[Platform1- OA-1 \nDQ]
                3.2[Platform1 - OA-2 N \nDQ]
                3.3[Platform1 - OA-N \n DQ]
            end
            AirByte-TiggerSync -->Transient-Location
            Transient-Location -->DQ         
            DQ -->6
            6{All-OK ?}
            7[Move]:::module
            8[Transient \n Files]:::module
            9[DBT-External \n Table Refresh]:::module
            10[DBT- SF\n JOBS]:::small-font
            S3[S3 - Bucket]:::module
            
        end
    end
    
    classDef module stroke:#0f0,font-size:9pt;
    classDef small-font font-size:9pt;
    class DQ,2.1,2.2,2.3 module
  
    6-->|Y|7
    7-->8
    9-->10
    G-->Typical-DAG
    8-->S3
    S3-->9
    
```
### SnowFlake Changes
- The components colored in green are areas where design and behavior of the existing system will be altered 
- For S3 bucket and file naming convention; please see section Airflow for the details
- Files in S3 don't appear automatically on External Tables in SnowFlake
- Simplest option is run an alter statement which refresh the external table meta data that has file listing 
- An elegant option is to set up AWS SQS [ Simple Queuing Service] that SF can subscribe to
- External Table will be partitioned based on year, month and date of the file or/and path through column expressions 
- External tables are likely to will be slow,i.e. relative to querying against tables with SF managed storage, this can be compensated thru materialized views if needed, the latency increase is small price to pay to have a de-coupled system
- S3 File format is open, Json being on top of the choice list , alternate formats that may be considered are parquet, csv in that order

```mermaid
flowchart
    subgraph S3-Bucket
        A[S3 -Bucket Name \n `with the full path`]
        subgraph Json-FileStructure
            Col1[co1: airbyte_id]
            Col2[col2 :airbyte_date_time]
            Col3[col3 :raw_data]
            Col4[col4 :additional_cntx \n optional]
        end
    end
    subgraph Connectivity
        X[AWS SQS \n OR \n  SnowFlake-Task]
    end

    subgraph SF
        1[External \n Stage]
        2[External \n Table]
        3[ Partitioned \n Column Expressions \n $FilePath]
        4[Optional \n Materialized Views]
    end
    classDef module stroke:#0f0
    A--> Json-FileStructure
    1-->2
    2-->3
    3-->4
    S3-Bucket-->Connectivity
    Connectivity-->SF
    class S3-Bucket,Connectivity,SF module

```

## Flow Diagram

####  OP Center Flow 

* UX -- Layer that facilitates Operational aspects of the system
* Service API -- Provides a nice decoupled architecture to interact with AirByte
* OpManager -- Backend Service as an AirFlow Job - core system functioning, logs maintenance, diagnostics,   measurements and metrics collection 
* Storage -- Layer that provides state-management for OPCenter

```mermaid
graph LR
    subgraph OPCenter
    OC.A[UX]-->OC.B[Service API]
    OC.B -->OC.C[Manager]
    OC.C -->OC.D[Storage] 
    OC.B -->OC.E[AirByte Service Link]
    OC.E -->OC.F[ Air Byte System]
end
```
#### AirFlow Vendor Platform Specific Flow

* For a give source platform, the DAG's in airflow will be instrumental to control flow  of business data
* A DAG flow may contain more than 1 Airbyte connection, for example multiple operator accounts could constitute a successful run, in such case, a DAG will encapsulate the flow rules
* Data Quality Module will instrument all flow for quality aspect of source data -- low count, no count, failed responses etc
* Upon successful Flow Run data will flow into AWS S3 as Json Files
``` mermaid
graph LR
    subgraph AirFlow Flow
        0[AirByte Service Link]
        1[Operator Account #1 ..N]
       
        5[Data Quality Framework]
        6[Gating Process]
        7{Success/Failure}
        8{Success}
        9{Failure}
        10(S3 Json)
        0-->1
        1-->5
        5-->6
        6-->7
        7-->8
        7-->9
        9-->|job failure|0
        8-->10
end
```
#### AirByte Platform Specific Flow

* AirByte will continue to be the platform of choice to configure  source  assets 
* AirByte connections will go thru Reusable Component Modules to strictly adhere to YML model based connections
* Airflow will "Always" host connections in manual model, external service will trigger designed connections
* Minimal to no interaction with AirByte for business , dev-ops will configuration , setup and  tuning 1 time
* Op-Center provides the interactivity with AirFlow for business users


``` mermaid
graph TB
    subgraph AirByte/DAG 
        AB.A[Connections]-->AB.B[Sources]
        AB.A-->AB.C[Platform Framework]
        AB.D[AirFlow Call]-->AB.A
        AB.D -->AB.E(AWS S3 Json)
end

```
#### SnowFlake Medallion Flow

* SnowFlake Raw Zones will be externalized via SF External Tables
* Existing DBT Jobs should continue to Function as today 
* No Changes to downstream processes in Phase-I beyond RaW/Bronze

``` mermaid
graph TB
    subgraph SnowFlake
        SF.A[S3 Json]-->SF.B[Bronze/RAW \n External Tables]
        SF.B-->SF.C(DBT Process 1)
        SF.C-->SF.D[SF Silver]
        SF.D -->SF.E(DBT Process 2)
        SF.E --> SF.F[ SF Gold ]
end

```

## Sequence Flow
- User Flow On Op-center
- Similar Flow is scheduled nightly in airbyte for OpCenter Manager to do similar connectivity checks
```mermaid
sequenceDiagram
    Actor User
    participant OpCenter-UX
    participant OpCenter-API
    participant AirByte
    User->>OpCenter-UX: AirByte Management
    activate OpCenter-UX
    OpCenter-UX ->>OpCenter-API: Request
    activate OpCenter-API
    OpCenter-API->>AirByte:Validate
    Activate AirByte
    AirByte-->OpCenter-API:Response
    deactivate AirByte
    OpCenter-API-->OpCenter-UX:Response
    deactivate OpCenter-API
    OpCenter-UX-->User:Source Status ?
    deactivate OpCenter-UX
    User->>OpCenter-UX: Logs & Diagnostics | Platform-Operator Grouping
    activate OpCenter-UX
    OpCenter-UX->>OpCenter-API:Request
    activate OpCenter-API
    OpCenter-API->>OpCenter-Storage:Request
    activate OpCenter-Storage
    OpCenter-Storage->OpCenter-API:Response
    deactivate OpCenter-Storage
    OpCenter-API->OpCenter-UX:Response
    deactivate OpCenter-API
    OpCenter-UX->User:Response
    deactivate OpCenter-UX
    OpCenter-Manager->>OpCenter-API:Request
   
```
## DQ Framework
The Design around DQ is still evolving, at the very core the DQ subsystem is likely to contain the following modules

- Metrics Computation:
    - Profiles leverages Analyzers to analyze each column of a dataset.
    - Analyzers serve here as a foundational module that computes metrics for data profiling and validation at scale.
- Constraint Suggestion:
    - Specify rules for various groups of Analyzers to be run over a dataset to return back a collection of constraints suggested to run in a Verification Suite.
- Constraint Verification:
    - Perform data validation on a dataset with respect to various constraints set by you.
- Metrics Repository
    - Allows for persistence and tracking of DQ runs over time.  

#### Phase-I DQ Scope

- Black box DQ 
  * Record Count
  * Daily Minimums 
  * Daily variance from historical averages

#### DQ Component Model
```mermaid

flowchart BT
  subgraph Data
    A[DataSet # 1]
    B[DataSet # 2]
  end
  subgraph DQ-Framework
    subgraph Rules
        C1[Data Quality Constraitns]
        C2(Constraint Suggestion)
        C3(Constraint Verification)
    end

    subgraph Validators
        D1[Profilers]
        D2[Analyzers]
        D3[Checks]
    end

    subgraph Metrics
        E1[Repository]
        E2[AWS S3]
        E3[In-Memory]
    end
  end
  subgraph Reports
    F1[Reports]
  end

  Validators-->Rules
  Rules-->Metrics
  Data-->Validators
  Metrics-->Reports


```
#### DQ-Data-Model

The conceptual Data Model here captures the core structures of the system. These structures hold state for the system when it is put to operate. The state of the system resides in these blocks/structures that help detail how the system is performing.  

The system state is comprised of 

- DQ State , Consists of reference tables and Data Quality Checks on Job that were run

- Job State, Captures Data From AirByte Runs





``` mermaid
erDiagram
    %%{init: { "theme": "forest" } }%%
    DATA_SOURCE {
        int id
        int account_id
        string platform_name
        string source_name 
        string path
        timestamp created_at
        timestamp last_updated
    }
     DATA_SOURCE_DQ{
        int id
        int data_source_id
        int data_quality_rule_id
        timestamp created_at
        timestamp last_updated
    }

    DATA_SOURCE_ITEM {
        int id
        int job_id
        int job_detail_id
        int data_source_id
        string item_ref
        string item_name
        string status
        timestamp created_at
        timestamp last_updated_at
    }

     DATA_QUALITY_DIMENSION {
        int id
        string name
        string description
    }
   
    DATA_QUALITY_RULE {
        int id
        int data_quality_dimension_id
        string name
        string description
        string expression
    }
    DATA_QUALITY_ISSUE {
        int id
        int data_quality_run
        int data_quality_rule_id
        int data_source_item_id
        string description
        string severity
        string status
    }
    PLATFORM{
        int id
        string name

    }
    ACCOUNT{
        int id
        int platform_id
        string name
        timestamp last_updated
        string username
        string password
        timestamp last_updated
    }

    JOB{
        int id
        string job_type
        int operator_id
        timestamp created_at
        timestamp updated_at
        string status
        int records_loaded
        timestamp last_updated
    }

    JOB_DETAIL {

        int id
        int job_id  
        string attempt_id
        string status 
        int records_loaded
        timestamp last_updated
    }


    DATA_SOURCE||--|{DATA_SOURCE_ITEM:"contains dataitems"
    DATA_QUALITY_RULE||--|{DATA_QUALITY_ISSUE:"specifies the constraint"
    DATA_QUALITY_DIMENSION||--|{DATA_QUALITY_RULE:"specifies the type of rule"
    JOB||--||DATA_SOURCE_ITEM:"produces"
    JOB_DETAIL||--||DATA_SOURCE_ITEM:"details the source"
    JOB||--|{JOB_DETAIL:"details the job"
    DATA_QUALITY_ISSUE||--|{DATA_SOURCE_ITEM:"ties it to specific source item"
    ACCOUNT||--|{DATA_SOURCE:"provides"
    PLATFORM||--|{ACCOUNT:"configures"
    DATA_QUALITY_RULE||--|{DATA_SOURCE_DQ:"enforces the quality"
    DATA_SOURCE||--|{DATA_SOURCE_DQ:"constraints"
  
  

```

#### DQ Data Flow

The main flow consists of 3 stages with airflow as the container that orchestrates these 3 stages

- STG1 is extraction where source datasets are fetched via pre-configured airbyte connectors

- STG2 expands the sourced batch data into individual transactional files

- STG3 performs the Data Quality Checks 

- STG4 not shown here wires the outputs from STG3 into SF, with AirFlow DBT tasks that finally conclude the run


``` mermaid
flowchart TD;

  
            subgraph "STG1-AIRBYTE"
                direction LR
                stg1.A[AirByte OPA TASK]
                stg1.B[S3 Folder \n Temp];
            end
            stg1.A-->|outputs|stg1.B
            subgraph "STG2-OPCENTER"
                direction LR
                stg2.A[AirByte \n REST API]
                stg2.B[ OPCenter \n DB \n Job/Job_Detail]
                stg2.C[File  \n Processor]
                stg2.D[TX \n Files]
                stg2.E[S3 Mirror]
                stg2.F[OPCenter \n DB \n Data Source Items ]
                stg2.A -->|Job/Job-Detail|stg2.B
                stg2.B-->|S3 Temp $ OPA Path|stg2.C
                stg2.C-->| Break Into |stg2.D
                stg2.D-->| Write into | stg2.E
                stg2.E-->| Queue Files for DQ|stg2.F
            end
      
        subgraph "STG3-DQ"
            direction LR
            stg3.A[ OPCenter \n DB]
            stg3.B[Data-Source \n Item/Items]
            stg3.C[Data \n Quality Rule/Rules]
            stg3.D{Decision}
            stg3.E[Raise \n Data \n Data Quality Issue]
            stg3.F[Move Data \n From S3 Mirror to \n Final S3]
            stg3.A-->|Fetch|stg3.B
            stg3.B-->|apply rules|stg3.C
            stg3.C-->|results|stg3.D
            stg3.D-->|failure| stg3.E
            stg3.D-->|success| stg3.F
            
        end
    STG1-AIRBYTE --->| JOB-ID |STG2-OPCENTER
    STG2-OPCENTER-->| JOB-ID|STG3-DQ
    
```

### Recovery Data Flow

The recovery flow is an additional DAG in airflow that leverages the main flow with the exception of STG1 where the platform connector runs in a recovery mode instead of a normal/scheduled run.

- The Airbyte source connector in the recovery mode relies on a key input, dates to be recovered  

- The connector is passed a list of dates { zero count days} from DQ state  

- The Airbyte internal methods over-ride the default chunking process, i.e. specific dates to source the payloads

- The recovered data flows into the same configured paths as the main process

``` mermaid
flowchart LR

    subgraph "Recovery Flow"
        direction LR

        subgraph "STG1"
            direction LR
            r.stg1.A[AirByte Recovery \n Connector]
            r.stg1.B[ S3 Folder \n Temp]
            r.stg1.A -->r.stg1.B
        end
        subgraph "STG2"
        r.stg2.A[STG2-OPCENTER]
        end

        subgraph "STG3"
        r.stg3.A[STG3-DQ]
        end

        STG1 --> STG2 --> STG3

    end

```

### Recovery Mock Data Snapshot

![alt text](recovery.png)


## Deployment Model

- Spike data platform will be baed on Kubernetes Stack
- Kubernetes will be designed in a way to set the node group to auto-scale
- Few Pods like ; AirFlow Scheduler; Op-Center Console will have persistent pods
- Pods will be short lived, the ones that are related to Airflow and AirByte Tasks

- An Abstract End Deployment Model

```mermaid
graph TD
    subgraph Kubernetes
        subgraph Compute Cluster
            A1[$Amazon EKS \n Control Plane]
            A2[$$Node Group \n AutoScale ]

        end

        subgraph Pods-Runtime
            B1(AirFlow Console \n Scheduler)
            B2(Meta Data \n Store)
            B3(OpCenter \n Console)
            subgraph Ephimeral-Pods
                C1(DAG Tasks)
                C2(AirByte Tasks)
                C3( Other Computes/ AI Tasks )

            end
        end
    end
    subgraph External-Systems
        D1(AWS - S3)
        D2(Other Data Stores)
        D3(SnowFlake)
    end

  A1-->|Manages| A2
  A2 -->|Fixed| B1
  A2-->|Fixed|B3
  A2-.->Ephimeral-Pods
  B3-->B2
  B1-->B2
  Ephimeral-Pods-->B2
  Pods-Runtime -->External-Systems

```
