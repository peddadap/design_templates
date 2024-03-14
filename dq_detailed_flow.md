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
