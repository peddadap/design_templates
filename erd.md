``` mermaid
erDiagram

    DATA_SOURCE {
        int id
        int operator_id
        string platform_name
        string operator 
        string account_id 
        string stream_name 
        string path
        timestamp created_at
        timestamp last_updated
    }

    DATA_SOURCE_ITEM {
        int id
        int job_id
        int data_source_id
        string item_name
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
   
    OPERATOR{
        int id
        string name
        timestamp last_updated
        string username
        string password
        int views
        int clicks
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
    JOB||--|{JOB_DETAIL:"references"
    DATA_QUALITY_ISSUE||--|{DATA_SOURCE_ITEM:"references"
    OPERATOR||--|{DATA_SOURCE:"references"

```


