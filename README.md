# kibana-audit-objects

**Intro**

This repo contains instructions on resolving the saved object id's of objects in Kibana with auditing logs for end user access logging.

The guide assumes the following:
1. A monitoring cluster with Kibana auditing data exists 
2. Relevent permissions to access system indices in the cluster 


**Set-up - Main Cluster**
- files contained in cluster-side folder

1. In Dev Tools (or via the API), create an ingest pipeline using the config from ingest-pipeline.txt.
This ingest pipeline extracts the saved objet id from the "_id" field of documents in the kibana_analytics index and removes fields that are not required for analysis. 

2. Create a componennt template using the component-template.txt file. This template contains the mapping for the new kibana objects index and speicifes use of the ingest pipeline.

3. Create an index template that uses the component template:

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

4. Reindex the existing kiban_analytics data into the new kibana index with the formlised mappings and ingest pipeline:

```
POST _reindex
{
  "source" : {
    "index" : ".kibana_analytics_8.11.0"
  },
  "dest" : {
    "index" : "kibana_objects_01",
    "pipeline" : "kibana-objectid"
  }
}
```

5. Add an advanced watch using the watcher.txt file. This watcher checks for new documents in the kibana_analytics index and reindexes into the new kibana index if the condition is met.
- You'll need to create an Api Key for authorization of the request.



**Set-up - Monitoring Cluster**
- files contained in mon-cluster-side folder

6. In the monitoring cluster, set up cross-cluster replication and create a follower index for the kibana objects index in the main cluster:

```
PUT /kibana_objects-01/_ccr/follow
{
  "remote_cluster": "mon-cluster",
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

7. Create a data view and verify the data in the monitoring cluster reflects that of the kibana-objects index in the main cluster.

The "kibana.saved_object.id" field should now exist in your audit logs and new object index - to be used in visualisations and enrichment policies. 

7. Create an enrich processor using enrich.txt and execute:

```
PUT _enrich/policy/objectid-policy/_execute
```

8. Create an ingest pipeline using ingest-pipeline.txt. This utilises the new enrichment policy and set processors to set the title of the object.

9. Crete a component template to formalise mappings of the new enriched index that will be used for visusalisations using, component-template.txt

10. Create an index template using the new component template:

```
PUT _index_template/trasform-objects
{
  "index_patterns": [
    "trasform-objects-*"
  ],
  "composed_of": [
    "transform-obj"
  ]
}
```

11. Create a transform from transform.txt and activate. the transform filters data from the kibana logs based on the presence of the saved_object.id field.

12. Create a watch that re-executes the enrich policy when new objects are added, using watcher.txt.

13. Import the export.ndjson file containing the configuration of the dashboard.

















