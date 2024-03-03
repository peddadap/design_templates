``` mermaid
flowchart LR;

        subgraph "DAG-1"
            direction LR
            subgraph "AIR-BYTE"
                direction LR
                A[AirByte OPA TASK]
                B[S3 Folder \n Mirror];
            end
            A-->|outputs|B
            subgraph "JOB-META-DATA"
                direction LR
                C[MYSQL \n DATA_SOURCE_ITEM];
                D[MYSQL \n JOB];
                E[MYSQL \n JOB_DETAIL];
            end
            D-->E-->C
        end
        AIR-BYTE-->JOB-META-DATA
    
        subgraph "DAG-2"
            direction LR
            G['Job_Id']-->|Job Cntx| H[Fetch \n Data \n Source Item/Items]
            H-->I[Apply \n Data \n Quality Rule/Rules]
            I-->|result| J{Decision}
            J-->|failure| K[Raise \n Data \n Data Quality Issue]
            J-->|success| L[Move Data \n From Mirror to \n Final S3]
            

        end

    DAG-1-->X('XCOM')
    X-->DAG-2
    




```