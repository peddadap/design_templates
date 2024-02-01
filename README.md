
# Software Design 

The design here incorporates  areas that are either being reengineered or extended to enhance the current functionality of several systems  
at spike up, most noticably airbyte , airflow and snowflake and with a brand new system for operational mangement  

## Table of Contents

- [Overview](#overview)
- [System Architecture](#system-architecture)
- [Flow Diagram](#flow-diagram)
- [Sequence Diagram](#sequence-flow)
- [Data Quality](#dq-framework)
- [Deployment Model](#dq-model)
- [Contributing](#contributing)
- [License](#license)

## Overview

![alt text](./spike_overview.jpg)

#### Core Systems as part of overall platform

- AirByte  
    * Align connector development around low code framework like  Airbyte YAML CDK
    * A path to decomission current source connectors that have significant code foot print 
    * A second level of generalization around YAML connectors on common operations like custom authnetication, specialized request construction  
      and wide format response parsing
    * Component registry build and deployment for resuability

- AirFlow
    * Re-engineered DAG's to improve system utilization by collating and parallelizing platform operations 
    * Stage - I Data Quality inspection and control flow of Airflow jobs
    * Better Diagnostics and Error reporting of flows on AirFlow

- Op Control Center

    * Decouple Business Op activity from airbyte connector framework 
    * Data Source Management --  new connections, reapir or fix connectivity and maintaince of source data assets 
    * Dedicated OP control center with near realtime monitoring and activity logs

- S3 RAW Zone

    * All sourced datasets will be stored on AWS S3 as parqauet files
    * Sources extracted will be partitioned by time and/or with other context appropriate for optimal downstream consumption

- SnowFlake

    * Externalize Raw Sources to cheaper alternative like S3
    * Phase-I, source datasets will appear as external tables inside SnowFlake
    * Phase II and beyond will engineer Snowflake for optimal compute and storage
 
    
### AirByte Changes

The components colored in green are areas where design and behavior of the existing system will be altered 

```mermaid
flowchart LR
   A1(Source \n Rest API)
    subgraph AirByte-Jobs
        subgraph Connections-yML 
            subgraph Alt-Connectors-1
                A2(Source \n Connector-1)
                A2.1(Python \n Modules):::module
            end
            A3(Connection)
            subgraph Alt-Connectors-2
                A4(Destination \n Connector)
                A4.1(Python \n Module):::module
            end
            A5(Transient \n Destination):::module
        end
    end
    classDef module stroke:#0f0
    A1-->AirByte-Jobs
    Alt-Connectors-1-->A3
    A3-->Alt-Connectors-2
    A4-->A5


```
### AirFlow Changes

The components colored in green are areas where design and behavior of the existing system will be altered 

```mermaid
flowchart LR
   subgraph "AirFlow"
        G(Scheduler)
        subgraph "Typical-DAG"
            1[Connection -1]
            2[DQ-CHK-1]:::module
            3[Connection -2]
            4[Connection -3]
            5[DQ-CHK-2]:::module
            6[Move]:::module
            7[Transient \n Destination]:::module
        end
    end
    
    subgraph "S3"
        S3.1(Connection 1 \n # S3):::module
        S3.2(Connection 2 \n # S3):::module
        S3.3(Connection 3 \n # S3):::module
    end
    classDef module stroke:#0f0
    1--> 2
    2 --> 3
    2 --> 4
    3--> 5
    4-->5
    5-->6
    6-->7
    G-->Typical-DAG
    AirFlow-->S3

```
### SnowFlake Changes
The components colored in green are areas where design and behavior of the existing system will be altered 


```mermaid
flowchart
    subgraph S3-Bucket
        A[Connection \n Name]
        B[Year]
        C[Month]
        D[Date]
        subgraph Parquet-File
            Col1
            Col2
            Col3
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
    A-->B
    B-->C 
    C-->D
    D-->Parquet-File
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
#### AirFlow Vendor Plaform Specific Flow

* For a give source platform, the DAG's in airflow will be instrumental to control flow  of business data
* A DAG flow may contain more than 1 Airbyte connection, for exaple multiple operator accounts could constitute a successful run, in such case, a DAG will encapsulate the flow rules
* Data Quality Module will instrument all flow for quality aspect of source data -- low count, no count, failed resonses etc
* Upon successful Flow Run data will flow into AWS S3 as Parquet Files
``` mermaid
graph LR
    subgraph AirFlow Flow
        0[AirByte Service Link]
        1[DataSet #1]
        2[DataSet #2]
        3[DataSet #3]
        4[DataSet #4] 
        5[Data Quality Framework]
        6[Gating Process]
        7{Success/Failure}
        8{Success}
        9{Failure}
        10(S3 Parquet)
        0-->1
        1--> 2
        1--> 3
        2--> 4
        3--> 4
        4-->5
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
* AirByte connections will go thru Resuable Component Modules to strictly adhere to YML model based connections
* Airflow will "Always" host connections in manual model, external service will trigger designed connections
* Minimal to no interaction with AirByte for bussiness , dev-ops will configuration , setup and  tuning 1 time
* OpCeneter provides the interactivity with AirFlow for business users


``` mermaid
graph TB
    subgraph AirByte/DAG 
        AB.A[Connections]-->AB.B[Sources]
        AB.A-->AB.C[Platform Framework]
        AB.D[AirFlow Call]-->AB.A
        AB.D -->AB.E(AWS S3 Parquet)
end

```
#### SnowFlake Medallion Flow

* SnowFlake Raw Zones will be exteranalized via SF External Tables
* Existing DBT Jobs should continue to Function as today 
* No Changes to downstream processes in Phase-I

``` mermaid
graph TB
    subgraph SnowFlake
        SF.A[S3 Parquet]-->SF.B[Bronze/RAW]
        SF.B-->SF.C(DBT Process 1)
        SF.C-->SF.D[SF Silver]
        SF.D -->SF.E(DBT Process 2)
        SF.E --> SF.F[ SF Gold ]
end

```

## Sequence Flow
- User Flow On Opcenter
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
There are 4 main components of Any DQ System, and they are:

- Metrics Computation:
    - Profiles leverages Analyzers to analyze each column of a dataset.
    - Analyzers serve here as a foundational module that computes metrics for data profiling and validation at scale.
- Constraint Suggestion:
    - Specify rules for various groups of Analyzers to be run over a dataset to return back a collection of constraints suggested to run in a Verification Suite.
- Constraint Verification:
    - Perform data validation on a dataset with respect to various constraints set by you.
- Metrics Repository
    - Allows for persistence and tracking of Deequ runs over time.
```mermaid

graph TD
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

  Rules-->Validators
  Metrics<-->Rules
  Data-->Validators
  Reports<-->Metrics


```

## Deployment Model

- Spike data platform will be baed on Kubernetes Stack
- Kubernetes will be designed in a way to set the node group to auto-scale
- Few Pods like ; AirFlow Scheduler; Op-Center Cosole will have persistant pods
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

## Contributing

[Include guidelines for others to contribute to your software design. This could include information about design reviews or contributions.]

## License

[Specify the license under which your software design is released.]

Feel free to customize this template based on your specific software design and how you want to represent it using Mermaid.js. Include more diagrams or details as needed for your project.