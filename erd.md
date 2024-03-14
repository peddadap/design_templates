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

    DATA_SOURCE||--|{DATA_SOURCE_ITEM:"references"
    DATA_QUALITY_RULE||--|{DATA_QUALITY_ISSUE:"references"
    DATA_QUALITY_DIMENSION||--|{DATA_QUALITY_RULE:"references"
    JOB||--||DATA_SOURCE_ITEM:"references"
    JOB_DETAIL||--||DATA_SOURCE_ITEM:"references"
    JOB||--|{JOB_DETAIL:"references"
    DATA_QUALITY_ISSUE||--|{DATA_SOURCE_ITEM:"references"
    ACCOUNT||--|{DATA_SOURCE:"references"
    PLATFORM||--|{ACCOUNT:"references"
    DATA_QUALITY_RULE||--|{DATA_SOURCE_DQ:"references"
    DATA_SOURCE||--|{DATA_SOURCE_DQ:"reference"
  
  

```


