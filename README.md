# kibana-audit-objects

**Intro**

This repo contains instructions on the full set of activities to complete the User Activity Monitoring Play
The approach is broken into Kibana Auditing and Elasticsearch Auditing and documented in such a way that either or both can be delivered.
* see the reference drawing UserActivityMonitoring_v3.excalidraw (open at www.excalidraw.com) for an overview

The guide assumes the following:
* A Platinum or Enterprise Subscription level (https://www.elastic.co/subscriptions) - see rows: Elasticsearch audit logging, Kibana audit logging
* Relevent permissions to access system indices in the cluster 

*** Create and Configure Monitoring Deployment ***
For Reference if needed: (https://www.elastic.co/guide/en/cloud/current/ec-monitoring-setup.html)

1.  Create (or make use of) a deployment that will be used for monitoring your production deployment. We'll refer to this as our monitoring deployment or cluster.

    To begin sending monitoring data from the production deployment to the monitoring deployment.

      -> Edit your production deployment, navigate to: 
      Monitoring -> Logs and metrics, and click Enable under "Ship to a deployment". 
      Choose your monitoring cluster and click Save.

2. Enable audit logging for both Elasticsearch and Kibana in your production deployment.
    
    -> **Elasticsearch**
    - Edit your deployment and add the following setting within *Manage user settings and extensions* for Elasticsearch: 
    
        ```
        xpack.security.audit.enabled: true
        xpack.security.audit.logfile.events.include: "authentication_success"
        xpack.security.audit.logfile.events.emit_request_body: true
        xpack.security.audit.logfile.events.ignore_filters.system.users: ["*_system", "found-internal-*",  "_xpack_security", "_xpack", "elastic/fleet-server","_async_search", "found-internal-admin-proxy"]
        xpack.security.audit.logfile.events.ignore_filters.realm.realms : [ "_es_api_key" ]
        xpack.security.audit.logfile.events.ignore_filters.internal_system.indices : ["metricbeat-*", "filebeat-*", "*ml-inference-native-*", "*monitoring-es-*"]
        ```
    
    -> **Kibana**
    - Continue editing your deployment and this time add the following setting within *Edit user settings* for Kibana:
        ```
        xpack.security.audit.enabled: true
        xpack.security.audit.ignore_filters:
        - categories: [web]
        - actions: [saved_object_open_point_in_time, saved_object_close_point_in_time, saved_object_find, space_get, saved_object_create]
        ```
    Save and restart the deployment for the new settings to take effect.

    -> **Validate**
    In the monitoring deployment, create a new Data View with the following settings: 

        - Name: elastic-cloud-logs
        - Index pattern: elastic-cloud-logs*
        - Timestamp field: @timestamp

**Set-up - Main Deployment**
- files contained in cluster-side folder


3. Set up the components and pipeline to create the kibana_objects_01 index which contains the object id to name mapping

    a) In Dev Tools (or via the API), create an ingest pipeline using the config from ingest-pipeline.txt.
    This ingest pipeline extracts the saved objet id from the "_id" field of documents in the kibana_analytics index and removes fields that are not required for analysis. 

    b) Create a component template using the component-template.txt file. This template contains the mapping for the new kibana objects index and speicifes use of the ingest pipeline.

    c) Create an index template that uses the component template:

        ```
        PUT _index_template/kibana_objects-new
        {
        "index_patterns": [
            "kibana_objects*"
        ],
        "composed_of": [
            "kibana-objects"
        ]
        }
        ```

    d) Reindex the existing kibana_analytics data into the new kibana index with the formlised mappings and ingest pipeline:

        ```
        POST _reindex
        {
        "source" : {
            "index" : ".kibana_analytics"
        },
        "dest" : {
            "index" : "kibana_objects-01",
            "pipeline" : "kibana-objectid"
        }
        }
        ```

4. Add an advanced watch using the watcher.txt file. This watcher checks for new documents in the kibana_analytics index and reindexes into the new kibana index if the condition is met.
- You'll need to create an Api Key for authorization of the request : Stack Management-> Security API Keys:

  ```
PUT _security/api_key
{
  "name": "reindex-watcher"
}
```

- You'll need to edit this by adding in the Elasticsearch (.es.) host name and API Key
- Note: When adding in the API key that line should start: "Authorization": "ApiKey xxxxxx"
- ... followed by a space and then the actual key value

**Set-up - Monitoring Cluster**
- files contained in mon-cluster-side folder

5. In the monitoring cluster, set up cross-cluster replication and create a follower index for the kibana objects index in the main cluster:

  a) Use Instructions here: https://www.elastic.co/guide/en/cloud/current/ec-configure-as-remote-clusters.html to set up a remote cluster named "main-cluster".  This is under Stack Management-> Data Remote Clusters, or via the API:

```
PUT _cluster/settings
{
  "persistent": {
    "cluster": {
      "remote": {
        "prod-cluster": {
          "skip_unavailable": false,
          "mode": "proxy",
          "proxy_address": "{{remote_host}}:{{remote_port}}",
          "proxy_socket_connections": 18,
          "server_name": "{{remote_host}}",
          "seeds": null,
          "node_connections": null
        }
      }
    }
  }
}
```

  b) Use this script to follow the kibana_objects-01 index from the main deployment.

    ```
    PUT /kibana_objects-01/_ccr/follow
    {
    "remote_cluster": "main-cluster",
    "leader_index": "kibana_objects-01",
    "max_read_request_operation_count": 5120,
    "max_outstanding_read_requests": 12,
    "max_read_request_size": "32mb",
    "max_write_request_operation_count": 5120,
    "max_write_request_size": "9223372036854775807b",
    "max_outstanding_write_requests": 9,
    "max_write_buffer_count": 2147483647,
    "max_write_buffer_size": "512mb",
    "max_retry_delay": "500ms",
    "read_poll_timeout": "1m"
    }
    ```
    
    -> **Validate**
    Create a data view and verify the data in the monitoring cluster reflects that of the kibana-objects index in the main cluster.

    The "kibana.saved_object.id" field should now exist in your audit logs and new object index - to be used in visualisations and enrichment policies. 

6. Create an enrich processor using enrich.txt and execute:
    a) Run the script in enrich.txt to create the processor
    b) Execute the processor

    ```
    PUT _enrich/policy/objectid-policy/_execute
    ```

7. Create an ingest pipeline using ingest-pipeline.txt. This utilises the new enrichment policy and set processors to set the title of the object.

8. Crete a component template to formalise mappings of the new enriched index that will be used for visusalisations.
    a) Use component-template.txt
    b) Create an index template using the new component template:

    ```
    PUT _index_template/kibana-transform
    {
      "template": {
        "settings": {
          "index": {
            "default_pipeline": "enrich-ids",
            "final_pipeline": "enrich-ids"
          }
        }
      },
      "index_patterns": [
        "kibana-transform-*"
      ],
      "composed_of": [
        "transform-obj"
      ]
    }
    ```

9. Create a transform from transform.txt and activate. The transform filters data from the kibana logs based on the presence of the saved_object.id field.
  a) Create the transform using transform.txt
  b) Start the transform
  
  ```
  post _transform/kibana-transform-01/_start

  ```

10. Create a watch that re-executes the enrich policy when new objects are added, using watcher.txt.
- You'll need to create an Api Key for authorization of the request : Stack Management-> Security API Keys:

PUT _security/api_key
{
  "name": "reindex-watcher"
}
```

- You'll need to edit this by adding in the Elasticsearch (.es.) host name and API Key
- Note: When adding in the API key that line should start: "Authorization": "ApiKey 
- ... followed by a space and then the actual key value

11. Import the ndjson files containing the configuration of the dashboards via Stack Management -> Saved Objects 

















