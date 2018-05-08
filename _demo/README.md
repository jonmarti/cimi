# Testing CIMI

To test CIMI, you'll need:

 - `docker-compose`

## Deploying CIMI with Docker

You'll need to provide CIMI over HTTPS. For this we use the
Traefik reverse proxy which can automatically request and renew
Let's Encrypt certificates, given that you have a FQDN.

For local testing however, self signed certificates can be used.

1. go to the `_demo` folder (where this README file sits)

2. generate the self signed certificates

`cd traefik/cert && openssl req  -nodes -new -x509  -keyout server.pem -out server.crt`

3. if using a FQDN, edit `traefik/traefik_https.toml`, changing the $$ placeholders and making sure `docker-compose.yml` points to it

4. Finally, run `docker-compose`

`docker-compose up`

Make sure that you've allocated enough memory to Docker; 4-6 GB of memory
should be sufficient (this is due to the temporary ES test backend).

After waiting some time for everything to start, you should be able to
see the CloudEntryPoint at the address `https://localhost/api/cloud-entry-point`.
You'll get the usual browser warning about the security of the certificates,
if you used the self-signed certificates for the server.

## Using CIMI

The first thing you'll want to do is to create a user so that you can login and manage your system resources.

### Regular User

1. create a regular user `testuser` with password `testpassword` (according to the `addRegularUser.json` file in this directory)

```bash
curl -XPOST -k -H "Content-type: application/json" https://localhost/api/user -d @addRegularUser.json
```

2. an email will be sent to you (if you haven't provided a SMTP server to CIMI, then **you might have to check your SPAM folder**). Copy the API address in that email, and paste it in your browser. Ex: `https://localhost/api/CALLBACK_ENDPOINT`.

3. login as `testuser`, using the `regularUser.json` session template in this directory

```bash
curl -XPOST -k https://localhost/api/session -d @regularUser.json -H 'content-type: application/json' --cookie-jar ~/cookies -b ~/cookies -sS -k
```

 #### API Keys

API keys are a safer way to have robots (scripts) interacting with CIMI on behalf of a user, since the same user can issue multiple API keys, and every one of them can be revoked without interfering with the original user access.

Basically CIMI distinguishes between _internal_ logins and _api_key_ logins, even though they might be associated with the same user account.

Before using API keys, create the session-template for it (only do it once):
```bash
curl -XPOST -H content-type:application/json -d '
{
   "method": "api-key",
   "instance": "api-key",

   "name" : "Login with API Key and Secret",
   "description" : "Authentication with API Key and Secret",
   "group" : "Login with API Key and Secret",

   "key" : "key",
   "secret" : "secret",

   "acl": {
             "owner": {"principal": "ADMIN",
                       "type":      "ROLE"},
             "rules": [{"principal": "ADMIN",
                        "type":      "ROLE",
                        "right":     "ALL"},
                       {"principal": "ANON",
                        "type":      "ROLE",
                        "right":     "VIEW"},
                       {"principal": "USER",
                        "type":      "ROLE",
                        "right":     "VIEW"}]
          }
}' https://localhost/api/session-template -k -H 'slipstream-authn-info: super ADMIN'
```

To create an API key, once you've logged in as `testuser` (step 3 above), do the following:

1. request the creation of an API key:

```bash
curl https://localhost/api/credential -X POST -H 'content-type: application/json' -d @generateAPIKey.json --cookie-jar ~/cookies -b ~/cookies -sS -k
```

you'll get something like

```json
{
  "status" : 201,
  "message" : "credential/4f8b8f66-2e15-4570-a14e-f9d3582425ad created",
  "resource-id" : "credential/4f8b8f66-2e15-4570-a14e-f9d3582425ad",
  "secretKey" : "nehrHa.V9Ppzb.vHf4BG.5vxv3j.DzLtqb"
}
```

2. take the key and secret:

```bash
export CIMI_API_KEY=credential/4f8b8f66-2e15-4570-a14e-f9d3582425ad
export CIMI_API_SECRET=nehrHa.V9Ppzb.vHf4BG.5vxv3j.DzLtqb
```

3. create another session login, `regularUserAPIKey.json`:

```bash
cat >regularUserAPIKey.json <<EOF
{
    "sessionTemplate": {
        "href": "session-template/api-key",
        "key": "$CIMI_API_KEY",
        "secret": "$CIMI_API_SECRET"
    }
}
EOF
```

4. login, same as testuser, but use `regularUserAPIKey.json` instead of `regularUser.json`



## Testing new changes

If you've added new resources to CIMI and want to test them:

1. build the project - go to the root folder and

`mvn clean install`

on the ouput you should see something like

```
[INFO] DOCKER> [mf2c/cimi-server:1.0.0-SNAPSHOT] "cimi-server": Created docker-build.tar in 171 milliseconds
[INFO] DOCKER> [mf2c/cimi-server:1.0.0-SNAPSHOT] "cimi-server": Built image sha256:596b0
[INFO] DOCKER> [mf2c/cimi-server:1.0.0-SNAPSHOT] "cimi-server": Removed old image sha256:d245c
[INFO] ------------------------------------------------------------------------
[INFO] Reactor Summary:
[INFO]
[INFO] server parent ...................................... SUCCESS [  0.762 s]
[INFO] server dep ......................................... SUCCESS [ 15.424 s]
[INFO] container .......................................... SUCCESS [  4.195 s]
[INFO] ------------------------------------------------------------------------
[INFO] BUILD SUCCESS
[INFO] ------------------------------------------------------------------------
```

2. find the docker-compose file

`cd _demo`

3. run it

`docker-compose up`

4. after a while you can find your new resources listed at `https://localhost/api/cloud-entry-point`

5. make a PR to merge your new changes to master

6. re-build everything in master, and push the generated Docker Image into the mF2C Registry

```bash
cd container
mvn -Ddocker.username=YOUR_DOCKER_USERNAME -Ddocker.password=YOUR_DOCKER_PWD docker:push
```

# Examples

## The "service" resource

After creating a session, the user can submit a **service** like this:

```bash
cat >>service.json <<EOF
{
  "name": "hello-world",
  "description": "Hello World Service",
  "resourceURI": "/hello-world",
  "exec": "hello-world",
  "exec_type": "docker",
  "exec_ports": [8080, 8081],
  "category": {
    "cpu": "low",
    "memory": "low",
    "storage": "low",
    "inclinometer": false,
    "temperature": false,
    "jammer": false,
    "location": false,
    "battery_level": true,
    "door_sensor": true,
    "pump_sensor": true,
    "accelerometer": true,
    "humidity": true,
    "air_pressure": true,
    "ir_motion": true
  }
}
EOF
```

Nothing is there yet:
`curl -XGET -k https://localhost/api/service --cookie-jar ~/cookies -b ~/cookies -sS -k`

```
{
  "count" : 0,
  "acl" : {
    "owner" : {
      "principal" : "ADMIN",
      "type" : "ROLE"
    },
    "rules" : [ {
      "principal" : "USER",
      "type" : "ROLE",
      "right" : "MODIFY"
    } ]
  },
  "resourceURI" : "http://schemas.dmtf.org/cimi/2/ServiceCollection",
  "id" : "service",
  "operations" : [ {
    "rel" : "add",
    "href" : "service"
  } ],
  "services" : [ ]
}
```

Let's submit the above service:
`curl -XPOST -k https://localhost/api/service -d @service.json -H 'content-type: application/json' --cookie-jar ~/cookies -b ~/cookies -sS -k`

```
{
  "status" : 201,
  "message" : "service/95e6eb84-b77a-42c1-822f-bf3523bdec2d created",
  "resource-id" : "service/95e6eb84-b77a-42c1-822f-bf3523bdec2d"
}
```
Double check it is there, and it is owned by the user:
`curl -XGET -k https://localhost/api/service --cookie-jar ~/cookies -b ~/cookies -sS -k`

```
{
  "count" : 1,
  "acl" : {
    "owner" : {
      "principal" : "ADMIN",
      "type" : "ROLE"
    },
    "rules" : [ {
      "principal" : "USER",
      "type" : "ROLE",
      "right" : "MODIFY"
    } ]
  },
  "resourceURI" : "http://schemas.dmtf.org/cimi/2/ServiceCollection",
  "id" : "service",
  "operations" : [ {
    "rel" : "add",
    "href" : "service"
  } ],
  "services" : [ {
    "description" : "Emergency Management System",
    "category" : {
      "cpu" : "low",
      "memory" : "low",
      "storage" : "low",
      "inclinometer" : true,
      "temperature" : true,
      "jammer" : true,
      "location" : true
    },
    "updated" : "2018-03-19T10:17:26.487Z",
    "name" : "EMS",
    "created" : "2018-03-19T10:17:26.487Z",
    "id" : "service/95e6eb84-b77a-42c1-822f-bf3523bdec2d",
    "acl" : {
      "owner" : {
        "principal" : "testuser",
        "type" : "USER"
      },
      "rules" : [ {
        "type" : "ROLE",
        "principal" : "ADMIN",
        "right" : "ALL"
      } ]
    },
    "operations" : [ {
      "rel" : "edit",
      "href" : "service/95e6eb84-b77a-42c1-822f-bf3523bdec2d"
    }, {
      "rel" : "delete",
      "href" : "service/95e6eb84-b77a-42c1-822f-bf3523bdec2d"
    } ],
    "resourceURI" : "http://schemas.dmtf.org/cimi/2/Service"
  } ]
}
```

Finally, the above service can be inspected at `curl -XGET -k https://localhost/api/service/95e6eb84-b77a-42c1-822f-bf3523bdec2d --cookie-jar ~/cookies -b ~/cookies -sS -k`


## The "sharing-model" resource

After creating a session,
```bash
cat >>sharingModel.json <<EOF
{
    "description": "example of a sharing model resource instance",
    "max_apps": 3,
    "gps_allowed": false,
    "max_cpu_usage": 1,
    "max_memory_usage": 1024,
    "max_storage_usage": 500,
    "max_bandwidth_usage": 20,
    "battery_limit": 10
}
EOF
```

Let's submit the above:
`curl -XPOST -k https://localhost/api/sharing-model -d @sharingModel.json -H 'content-type: application/json' --cookie-jar ~/cookies -b ~/cookies -sS -k`

Double check it is there:
`curl -XGET -k https://localhost/api/sharing-model --cookie-jar ~/cookies -b ~/cookies -sS -k`


## The "service-instance" resource

Same as the other ones above. The JSON resource should look like:
```bash
cat >>serviceInstance.json <<EOF
{
    "user": "test_user",
    "service": "a3bcf627-99cb-4138-a429-b44c5645e6d2",
    "agreement": "e6bcf627-99cb-4138-a429-b44c5645e6b5",
    "status": "running",
    "agents": [
        {
          "agent": {"href": "device/testdevice"},
          "ports": [8081],
          "container_id": "8259ec5ce1c97c0ce0e12e671f532b6fe44c015ee422ec10320df180a0e6da38",
          "status": "running",
          "num_cpus": 3,
          "allow": true,
          "master_compss": false,
          "agent_param": "-"
        }
    ]
}
EOF
```

## The "user-profile" resource

Same as the other ones above. The JSON resource should look like:
```bash
cat >>userProfile.json <<EOF
{
      "service_consumer": true,
      "resource_contributor": false
}
EOF
```


## The "agreement" resource

Same as the other ones above. The JSON resource should look like:
```bash
cat >>agreement.json <<EOF
{
    "id": "a02",
    "name": "Agreement 02",
    "state": "stopped",
    "details":{
        "id": "a02",
        "type": "agreement",
        "name": "Agreement 02",
        "provider": { "id": "mf2c", "name": "mF2C Platform" },
        "client": { "id": "c02", "name": "A client" },
        "creation": "2018-01-16T17:09:45.0Z",
        "expiration": "2019-01-17T17:09:45.0Z",
        "guarantees": [
            {
                "name": "TestGuarantee",
                "constraint": "[test_value] < 10"
            }
        ]
    }
}
EOF
```

## The "sla-violation" resource

Same as the other ones above. The JSON resource should look like:
```bash
cat >>slaViolation.json <<EOF
{
    "guarantee" : "TestGuarantee",
    "datetime" : "2018-04-11T10:39:51.527008088Z",
    "agreement_id" : "agreement/4e529393-f659-44d6-9c8b-b0589132599b"
}
EOF

```


## The "device" resource

Same as the other ones above. The JSON resource should look like:
```bash
cat >>device.json <<EOF
{
    "deviceID": "123",
    "isLeader": false,
    "os": "Linux-4.13.0-38-generic-x86_64-with-debian-8.10",
    "arch": "x86_64",
    "cpuManufacturer": "Intel(R) Core(TM) i7-8550U CPU @ 1.80GHz",
    "physicalCores": 4,
    "logicalCores": 8,
    "cpuClockSpeed": "1.8000 GHz",
    "memory": 7874.2109375,
    "storage": 234549.5078125,
    "powerPlugged": true,
    "networkingStandards": "['eth0', 'lo']",
    "ethernetAddress": "[snic(family=<AddressFamily.AF_INET: 2>, address='172.17.0.3', netmask='255.255.0.0', broadcast='172.17.255.255', ptp=None), snic(family=<AddressFamily.AF_PACKET: 17>, address='02:42:ac:11:00:03', netmask=None, broadcast='ff:ff:ff:ff:ff:ff', ptp=None)]",
    "wifiAddress": "Empty",
    "hwloc": "<xmlString>",
    "cpuinfo": "<rawCPUinfo>"
}
EOF

```


## The "device-dynamic" resource

Same as the other ones above. The JSON resource should look like:
```bash
cat >>deviceDynamic.json <<EOF
{
    "device": {"href": "device/123"},
    "ramFree": 4795.15234375,
    "ramFreePercent": 60.9,
    "storageFree": 208409.25,
    "storageFreePercent": 93.6,
    "cpuFreePercent": 93.5,
    "powerRemainingStatus": "30.75885328836425",
    "powerRemainingStatusSeconds": "3817",
    "ethernetAddress": "[snic(family=<AddressFamily.AF_INET: 2>, address='172.17.0.3', netmask='255.255.0.0', broadcast='172.17.255.255', ptp=None), snic(family=<AddressFamily.AF_PACKET: 17>, address='02:42:ac:11:00:03', netmask=None, broadcast='ff:ff:ff:ff:ff:ff', ptp=None)]",
    "wifiAddress": "Emp": [1595,8644,16,74,0,0,0,0],
    "wifiThroughputInfo": ["E","m","p","t","y"],
    "myLeaderID": {"href": "device/1"}
}
EOF

```

## The "fog-area" resource

Same as the other ones above. The JSON resource should look like:
```bash
cat >>fogArea.json <<EOF
{
    "leaderDevice": {"href": "device/123refegh"},
    "numDevices": 10,
    "ramTotal": 56789.90,
    "ramMax": 4569.34,
    "ramMin": 1478.34,
    "storageTotal": 120003456798.23456,
    "storageMax": 345678000.23456,
    "storageMin": 3456789.248,
    "avgProcessingCapacityPercent": 88.6,
    "cpuMaxPercent": 98.2,
    "cpuMinPercent": 56.7,
    "avgPhysicalCores": 4,
    "physicalCoresMax": 6,
    "physicalCoresMin":  2,
    "avgLogicalCores" : 4,
    "logicalCoresMax": 6,
    "logicalCoresMin": 2,
    "powerRemainingMax": "Device has unlimited power source",
    "powerRemainingMin": "88.2"
 }
EOF

```


## The "service-operation-report" resource

Same as the other ones above. The JSON resource should look like:
```bash
cat >>serviceOperationReport.json <<EOF
{
    "serviceInstance": {"href": "service-instance/asasdasd"},
    "operation": "newMethod",
    "execution_time": 123.32
}

EOF

```
