# Installation
```bash
docker pull sebp/elk
```
```bash
# 5601 Kibana
# 9200 Elasticsearch
docker run -p 5601:5601 -p 9200:9200 -e MAX_MAP_COUNT=262144 -it --name elk sebp/elk
```

# Shared index
## One index
```json
PUT data 
{
  "mappings": {
    "properties": { 
      "name":     { "type": "text"  }, 
      "country":     { "type": "text"  }, 
      "population": { "type": "integer" },
      "language": { "type": "keyword" },
      "tenantId": { "type": "keyword" }
    }
  }
}
GET data/_settings
GET data/_mapping
PUT data/_doc/1
{
    "name" : "Gdansk",
    "country" : "Poland",
    "population" : 70000,
    "language": "en",
    "tenantId": "abc"
}
PUT data/_doc/2
{
    "name" : "กดัญสก์",
    "country" : "ประเทศโปแลนด์",
    "population" : 70000,
    "language": "th",
    "tenantId": "abc"
}
```

## Tenant fields in one index
```json
DELETE data
PUT data 
{
  "mappings": {
    "properties": { 
      "name":     { "type": "text"  }, 
      "country":     { "type": "text"  }, 
      "population": { "type": "integer" },
      "language": { "type": "keyword" },
      "tenantId": { "type": "keyword" },
      "tenant_X_district": {"type": "text"}
    }
  }
}
PUT data/_doc/1
{
    "name" : "Gdansk",
    "country" : "Poland",
    "population" : 70000,
    "language": "en",
    "tenantId": "abc"
}
PUT data/_doc/2
{
    "name" : "กดัญสก์",
    "country" : "ประเทศโปแลนด์",
    "population" : 70000,
    "language": "th",
    "tenantId": "abc"
}
PUT data/_doc/3
{
    "name" : "Gdansk",
    "tenant_X_district": "Kokoszki",
    "country" : "Poland",
    "population" : 70000,
    "language": "en",
    "tenantId": "xyz"
}
GET data/_mapping
```

# Routing
```json
PUT _template/default
{
  "index_patterns": ["*"],
  "order": -1,
  "settings": {
    "number_of_shards": "3",
    "number_of_replicas": "1"
  }
}
DELETE data
PUT data 
{
  "mappings": {
    "properties": { 
      "name":     { "type": "text"  }, 
      "country":     { "type": "text"  }, 
      "population": { "type": "integer" },
      "language": { "type": "keyword" },
      "tenantId": { "type": "keyword" },
      "tenant_X_district": {"type": "text"}
    }
  }
}
PUT data/_doc/1?routing=abc
{
    "name" : "Gdansk",
    "country" : "Poland",
    "population" : 70000,
    "language": "en",
    "tenantId": "abc"
}
PUT data/_doc/3?routing=xyz
{
    "name" : "Gdansk",
    "tenant_X_district": "Kokoszki",
    "country" : "Poland",
    "population" : 70000,
    "language": "en",
    "tenantId": "xyz"
}
GET data/_search?routing=xyz
{
  "query": {
    "match": {
      "tenantId": "xyz"
    }
  }
}
```

# Index template
## Remove old data + config
```json
GET _template
DELETE _template/default
DELETE data
```
## Add index template and index documents
```json
PUT _template/data
{
  "index_patterns": ["*data_*"],
  "order": 0,
  "settings": {
    "number_of_shards": "3",
    "number_of_replicas": "1"
  },
  "mappings": {
    "properties": {
      "name":     { "type": "text"  }, 
      "country":     { "type": "text"  }, 
      "population": { "type": "integer" },
      "language": { "type": "keyword" },
      "tenantId": { "type": "keyword" }
    }
  }
}
PUT _template/data_big_tenant
{
  "index_patterns": ["*data_*_big_tenant"],
  "order": 1,
  "mappings": {
    "properties": {
      "tenant_X_district": {"type": "text"}
    }
  }
}

PUT data_en/_doc/1?routing=abc
{
    "name" : "Gdansk",
    "country" : "Poland",
    "population" : 70000,
    "language": "en",
    "tenantId": "abc"
}
PUT data_th/_doc/2?routing=abc
{
    "name" : "กดัญสก์",
    "country" : "ประเทศโปแลนด์",
    "population" : 70000,
    "language": "th",
    "tenantId": "abc"
}
PUT data_en_big_tenant/_doc/3?routing=xyz
{
    "name" : "Gdansk",
    "tenant_X_district": "Kokoszki",
    "country" : "Poland",
    "population" : 70000,
    "language": "en",
    "tenantId": "xyz"
}
GET data_en/_mapping
GET data_en_big_tenant/_mapping

GET data_en*/_search?routing=xyz
{
  "query": {
    "match": {
      "tenantId": "xyz"
    }
  }
}
```

# Index alias
## Remove old data (keep old index template)
```json
DELETE data*
```

## Add indexes in v1
```json
PUT v1_data_en/_doc/1?routing=abc
{
    "name" : "Gdansk",
    "country" : "Poland",
    "population" : 70000,
    "language": "en",
    "tenantId": "abc"
}
PUT v1_data_th/_doc/2?routing=abc
{
    "name" : "กดัญสก์",
    "country" : "ประเทศโปแลนด์",
    "population" : 70000,
    "language": "th",
    "tenantId": "abc"
}
PUT v1_data_en_big_tenant/_doc/3?routing=xyz
{
    "name" : "Gdansk",
    "tenant_X_district": "Kokoszki",
    "country" : "Poland",
    "population" : 70000,
    "language": "en",
    "tenantId": "xyz"
}

POST /_aliases
{
    "actions" : [
        { "add" : { "index" : "v1_data_en", "alias" : "data_en" } },
        { "add" : { "index" : "v1_data_th", "alias" : "data_th" } },
        { "add" : { "index" : "v1_data_en_big_tenant", "alias" : "data_en_big_tenant" } }
    ]
}
GET data_en*/_search?routing=xyz
{
  "query": {
    "match": {
      "tenantId": "xyz"
    }
  }
}
```

## Update configuration and add v2 indexes
```json
PUT _template/data
{
  "index_patterns": ["*data_*"],
  "order": 0,
  "settings": {
    "number_of_shards": "4",
    "number_of_replicas": "1"
  },
  "mappings": {
    "properties": {
      "name":     { "type": "text"  }, 
      "country":     { "type": "text"  }, 
      "population": { "type": "integer" },
      "language": { "type": "keyword" },
      "tenantId": { "type": "keyword" }
    }
  }
}

PUT v2_data_en/_doc/1?routing=abc
{
    "name" : "Gdansk",
    "country" : "Poland",
    "population" : 70000,
    "language": "en",
    "tenantId": "abc"
}
PUT v2_data_th/_doc/2?routing=abc
{
    "name" : "กดัญสก์",
    "country" : "ประเทศโปแลนด์",
    "population" : 70000,
    "language": "th",
    "tenantId": "abc"
}
PUT v2_data_en_big_tenant/_doc/3?routing=xyz
{
    "name" : "Gdansk",
    "tenant_X_district": "Kokoszki",
    "country" : "Poland",
    "population" : 70000,
    "language": "en",
    "tenantId": "xyz"
}
POST /_aliases
{
    "actions" : [
        { "remove" : { "index" : "v1_data_en", "alias" : "data_en" } },
        { "remove" : { "index" : "v1_data_th", "alias" : "data_th" } },
        { "remove" : { "index" : "v1_data_en_big_tenant", "alias" : "data_en_big_tenant" } },
        { "add" : { "index" : "v2_data_en", "alias" : "data_en" } },
        { "add" : { "index" : "v2_data_th", "alias" : "data_th" } },
        { "add" : { "index" : "v2_data_en_big_tenant", "alias" : "data_en_big_tenant" } }
    ]
}
GET data_en*/_search?routing=xyz
{
  "query": {
    "match": {
      "tenantId": "xyz"
    }
  }
}
```