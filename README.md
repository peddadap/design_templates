
# Software Design 

The design here incorporates  areas that are either being reengineered or extended to enhance the current functionality of several systems  
at spike up, most noticably airbyte , airflow and snowflake and with a brand new system for operational mangement  

## Table of Contents

- [Overview](#overview)
- [System Architecture](#system-architecture)
- [Flow Diagram](#flow-diagram)
- [Sequence Diagram](#sequence-diagram)
- [Installation](#installation)
- [Usage](#usage)
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
    * Phase II and beyond will engineer Snowflake for optimal computer and storage
 
    
## System Architecture

```mermaid

graph TB
  
  subgraph "Architectural Block"
    subgraph "Sources" 
      A(Source 1)
      B(Source 2)
      C(Source N)
    end

    subgraph "Connectors"
      D(Connector 1)
      E(Connector 2)
      F(Connector N)
      SC(Shared Components)
    end

    subgraph "DAG System"
      G(Scheduler)
      subgraph "Typical DAG"
        1[Task 1]
        2[Task 2]
        3[Task 3]
        4[Task 4]
        5[Task 5]
    end
    end

 
    subgraph "S3"
      I(Destination 1)
      J(Destination 2)
      K(Destination N)
    end
end

  A --> D
  B --> E
  C --> F
  D --> G
  E --> G
  F --> G

  G --> I
  G --> J
  G --> K
  1--> 2
  1 --> 3
  2 --> 4
  3 --> 4
  4 --> 5


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
        SF.A[S3 Parquet]-->SF.B[SF External Tables
                                 Bronze/RAW]
        SF.B-->SF.C(DBT Process 1)
        SF.C-->SF.D[SF Silver]
        SF.D -->SF.E(DBT Process 2)
        SF.E --> SF.F[ SF Gold ]
end

```

## Sequence Diagram

```mermaid
sequenceDiagram
  participant User
  participant System
  User->>System: Request
  activate System
  System-->>User: Response
  deactivate System
```

[Use Mermaid.js syntax to create a sequence diagram illustrating the interaction between different parts of your system.]

## Installation

[Provide instructions on how to install or set up your software design.]

## Usage

[Explain how to use your software design. Include examples or scenarios.]

## Contributing

[Include guidelines for others to contribute to your software design. This could include information about design reviews or contributions.]

## License

[Specify the license under which your software design is released.]

Feel free to customize this template based on your specific software design and how you want to represent it using Mermaid.js. Include more diagrams or details as needed for your project.