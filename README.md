# Repeat Mannys tests on 1.18.2 livedata-migrator-1.18.2-2906.noarch

These steps will replicate the scenario:
> tables are migrated to glue with an extra slash in the table location.
> Need to correct by reconfiguring the glue agent and resetting the migration??

1. Tables will be put into AWS using a glue agent with no supplied DefaultFS Override.
2. Result will be confirmed as data having the extra slash after the filesystem. e.g. s3a://jhugh-euwest**//**
3. Attempt to rectify the tables in glue by updating the glue agent with no trailing slash and migration reset.

## Environment.
### Confirm versions.( Only has the last patch hivemigrator-1.9.11-1057.noarch.rpm)
```
rpm -qa | grep -e "live\|hivemigrator"

hivemigrator-azure-hdi-1.9.6-1030.noarch
hivemigrator-1.9.6-1030.noarch
livedata-migrator-1.18.2-2906.noarch
livedata-ui-10.3.6-2980.noarch
livedata-migrator-cli-1.9.1-302.noarch
```
### Filesystem info(Note the fsType and fs.defaultFS for AWS.)
```
curl -X GET http://10.6.123.132:18080/fs/sources

[ {
  "fileSystemId" : "sourceHDFS",
  "fsType" : "hdfs",
  "isSource" : true,
  "properties" : {
    "fs.defaultFS" : "hdfs://nameservice02"
  },
  "eventsPosition" : 0,
  "isScanOnly" : false
} ]
```
```
curl -X GET http://10.6.123.132:18080/fs/targets
[ {
  "fileSystemId" : "targetAWS",
  "fsType" : "s3a",
  "isSource" : false,
  "properties" : {
    "fs.defaultFS" : "s3a://jhugh-euwest/"
  },
  "eventsPosition" : 0,
  "isScanOnly" : false
} ]
```

## 1. Replicate with a glue agent created with no DefaultFS Override supplied.

### Confirm no data in AWS
```
aws glue get-tables --database-name serdetest | grep STORAGEDESCRIPTOR | awk '{print $4}'
```
### Add AWS glue agent with **NO DefaultFS Override supplied**.
```
curl -X 'POST' 'http://10.6.123.132:6780/agents/glue' -H 'accept: application/json' -H 'Content-Type: application/json' -d '{"agentType": "GLUE","glueAgentConfig": {"storageIdentifier": "targetAWS","glueEndpoint": "vpce-01739c7bb68436cea-947aw4se.glue.eu-west-1.vpce.amazonaws.com","awsRegion": "eu-west-1","credentialsProvider": "com.wandisco.hivemigrator.agent.awsglue.auth.StaticCredentialsProviderFactory","accessKey": "REDACTED","secretKey": "REDACTED"},"fileSystemId": "targetAWS"}'
```

### Check agent details.
```
curl -X 'GET' 'http://10.6.123.132:6780/agents/glue' -H 'accept: application/json'
```

```
{
  "name" : "glue",
  "location" : "LOCAL",
  "config" : {
    "agentType" : "GLUE",
    "glueAgentConfig" : {
      "glueEndpoint" : "vpce-01739c7bb68436cea-947aw4se.glue.eu-west-1.vpce.amazonaws.com",
      "awsRegion" : "eu-west-1",
      "credentialsProvider" : "com.wandisco.hivemigrator.agent.awsglue.auth.StaticCredentialsProviderFactory",
      "accessKey" : "********************",
      "secretKey" : "****************************************",
      "glueMaxRetries" : 0,
      "glueMaxConnections" : 0,
      "glueMaxSocketTimeout" : 0,
      "glueConnectionTimeout" : 0,
      "storageIdentifier" : "vpce-01739c7bb68436cea-947aw4se.glue.eu-west-1.vpce.amazonaws.com:eu-west-1:null"
    },
    "fileSystemId" : "targetAWS",
    "defaultFsOverride" : "s3a://jhugh-euwest/"
  }
}

```

### Add migration rule: hivepatternrule00001
```
curl -X 'POST' 'http://10.6.123.132:6780/rule' -H 'accept: application/json' -H 'Content-Type: application/json' -d '{"ruleName": "hivepatternrule00001","dbNamePattern":"serdetest","tableNamePattern": "*"}'
```

### Create a migration: migtest14 with rule: hivepatternrule00001
```
curl -X 'POST' 'http://10.6.123.132:6780/migration' -H 'accept: application/json' -H 'Content-Type: application/json' -d '{"name": "migtest14","sourceAgentName":"source","targetAgentName": "glue","rules": ["hivepatternrule00001"]}'
```

### Start migration.
```
curl -X 'POST' 'http://10.6.123.132:6780/migration/start' -H 'accept: application/json' -H 'Content-Type: application/json' -d '{"migrationNames": ["migtest14"]}'
```

### Check data in AWS
```
aws glue get-tables --database-name serdetest | grep STORAGEDESCRIPTOR | awk '{print $4}'
```

```
s3a://jhugh-euwest//data/databases/serdetest/tables/table1
s3a://jhugh-euwest//data/databases/serdetest/tables/table2
s3a://jhugh-euwest//data/databases/serdetest/tables/table3
```

## 2. Fix attempt.


### Stop migration
```
curl -X 'POST' 'http://10.6.123.132:6780/migration/stop' -H 'accept: application/json' -H 'Content-Type: application/json' -d '["migtest14"]'
```

### UPDATE AWS glue agent with **WITH DefaultFS Override supplied**.
```
curl -X 'PUT' 'http://10.6.123.132:6780/agents/glue' -H 'accept: application/json' -H 'Content-Type: application/json' -d '{"agentType": "GLUE","glueAgentConfig": {"storageIdentifier": "targetAWS","glueEndpoint": "vpce-01739c7bb68436cea-947aw4se.glue.eu-west-1.vpce.amazonaws.com","awsRegion": "eu-west-1","credentialsProvider": "com.wandisco.hivemigrator.agent.awsglue.auth.StaticCredentialsProviderFactory","accessKey": "REDACTED","secretKey": "REDACTED"},"fileSystemId": "targetAWS","defaultFsOverride": "s3a://jhugh-euwest"}'
```

### Check agent details.
```
curl -X 'GET' 'http://10.6.123.132:6780/agents/glue' -H 'accept: application/json'
```

```
{
  "name" : "glue",
  "location" : "LOCAL",
  "config" : {
    "agentType" : "GLUE",
    "glueAgentConfig" : {
      "glueEndpoint" : "vpce-01739c7bb68436cea-947aw4se.glue.eu-west-1.vpce.amazonaws.com",
      "awsRegion" : "eu-west-1",
      "credentialsProvider" : "com.wandisco.hivemigrator.agent.awsglue.auth.StaticCredentialsProviderFactory",
      "accessKey" : "********************",
      "secretKey" : "****************************************",
      "storageIdentifier" : "vpce-01739c7bb68436cea-947aw4se.glue.eu-west-1.vpce.amazonaws.com:eu-west-1:null"
    },
    "fileSystemId" : "targetAWS",
    "defaultFsOverride" : "s3a://jhugh-euwest"
  }
}
```

### Reset migration
```
curl -X 'POST' 'http://10.6.123.132:6780/migration/reset' -H 'accept: application/json' -H 'Content-Type: application/json' -d '{"migrationNames": ["migtest14"],"forceStop": true}'
```

### Start migration.
```
curl -X 'POST' 'http://10.6.123.132:6780/migration/start' -H 'accept: application/json' -H 'Content-Type: application/json' -d '{"migrationNames": ["migtest14"]}'
```

### Check data in AWS
```
aws glue get-tables --database-name serdetest | grep STORAGEDESCRIPTOR | awk '{print $4}'
```
```
s3a://jhugh-euwest///data/databases/serdetest/tables/table1
s3a://jhugh-euwest///data/databases/serdetest/tables/table2
s3a://jhugh-euwest///data/databases/serdetest/tables/table3
```

# Reset environment.
### 1. Stop migration
```
curl -X 'POST' 'http://10.6.123.132:6780/migration/stop' -H 'accept: application/json' -H 'Content-Type: application/json' -d '["migtest14"]'
```
### 2. Remove migration.
```
curl -X 'DELETE'   'http://10.6.123.132:6780/migration/migtest14?stop=false'   -H 'accept: */*'
```
### 3. Remove agent.
```
curl -X 'DELETE'   'http://10.6.123.132:6780/agents/glue'   -H 'accept: application/json'
```

### 4. Remove migration rule: hivepatternrule00001
```
curl -X 'DELETE' 'http://10.6.123.132:6780/rule/hivepatternrule00001' -H 'accept: */*'
```

### 5. Delete tables from target.
```
aws glue delete-database --name serdetest
```

** NOW. DO a global replace on this file on your migration rule name to a new one. **
```
./increment.sh
```
currentMigrationVersion: 14