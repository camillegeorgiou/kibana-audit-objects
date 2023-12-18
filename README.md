# kibana-audit-objects

**Intro**

This repo contains instructions on resolving the saved object id's of objects in Kibana with auditing logs.
The guide assumes the following:
1. A monitoring cluster with Kibana auditing data exists 
2. Relevent permissions to access system indices in the cluster 


**Set-up**

1. In Dev Tools (or via the API), create an ingest pipeline using the config from ingest-pipeline.txt.
This ingest pipeline extracts the saved objet id from the "_id" field of documents in the kibana_analytics index and removes fields that are not required for analysis. 

2. Create a componennt template using the component-template.txt file. This template contains the mapping for the new kibana objects index and speicifes use of the ingest pipeline.

3. Create an index template that uses the component template:

PUT _index_template/kibana_objects-new
{
  "index_patterns": [
    "kibana_objects-new"
  ],
  "composed_of": [
    "kibana-objects"
  ]
}

4. Add an advanced watch using: watcher.txt. This watcher checks for new documents in the kibana_analytics index and reindexes into the new kibana index if the condition is met.

5. In the monitoring cluster, set up cross-cluster replication and create a follower index for the kibana objects index in the main cluster:

PUT /kibana_objects-9/_ccr/follow
{
  "remote_cluster": "{{remote cluster name}}",
  "leader_index": "kibana_objects-new,
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

6. Create a data view and verify the data in the monitoring cluster reflects that of the kibana-objects-new index in the main cluster.

The "kibana.saved_object.id" field should now exist in your audit logs and new object index - to be used in visualisations and enrichment policies. 











