# Telemetry API : Alert CRUD

Using the Telemetry Service, it is possible to create and maintain **alert definitions** : criteria around a metric
value that you specify for a particular deployment.  (For brevity, we will sometimes use "alert" as a synonym for "alert
definition" here.)

For each active alert definition, the Telemetry system will monitor the relevant metric values continuously, and will
generate notifications whenever that value series goes outside the specified criteria for the resource(s) in that deployment.

This document describes the API for programmatic control of these alert definitions via RESTful HTTPS endpoints.


### Client authentication : `Telemetry-API-Key` request header

Client authentication is required in the form of an API Key header that must be sent with every HTTPS request: `Telemetry-API-Key`.  

The server will verify the validity of the API key string found in this header value, confirm it exists in the set of currently authorized keys, and further verify that the authorization in question permits access to any deployment resources involved in the request or (where applicable) its response.


#### Create a new alert definition


```
POST https://telemetry-api.mongolab.com/loc/v0/alerts
```

A created alert definition, specified in the request body, has the following structure:

```
{
    _id: ID,
    tags: { 
        TAG_NAME : TAG_VALUE ,
        ... 
    },
    deployment: DEPLOYMENT_ID,
    filter: FILTER_ID,
    condition: { 
        metric: METRIC_ID,
        max: NUMBER,
        min: NUMBER
    },
}
```

* ```_id```: (optional) Unique identifier.  If not provided, one will be automatically created.  If provided, but it
  conflicts with an existing alert definition, an error will be returned.
* ```tags```: (optional) User-defined free-form fields.  Tag names must be alphanumeric, may contain hyphens ('-') or underscores ('_'), and must not start with a number.  A tag value can be any string.  A tag's name and value must each comprise fewer than 1024 characters.
* ```deployment```: Specifies the ID of the root deployment resource (e.g., replica set cluster) to which this alert definition is attached. All servers belonging to this deployment (subject to ```filter```, if given) will produce alert events based on the given ```condition```. 
* ```filter```: (optional) By default the alert will apply to the deployment and all its constituent resources. If the alert
only applies to a subset of those resources, a filter may be specified to narrow this scope. The set of available filters currently comprises:

    | `filter:` value | nodes selected |
    | --------------- | -------------- |
    | `"SERVER_ROLE_MONGOD_PRIMARY"` | Primary data node(s) |
    | `"SERVER_ROLE_CONFIG_MONGOD"` | Config servers |
    | `"SERVER_ROLE_MONGOS"` | Mongos routers |
    | `"SERVER_ROLE_MONGOD_DATA"` | Any data node |
    | `"SERVER_ROLE_MONGOD_SECONDARY"` | Secondary data node(s) |


* ```condition```: Describes the conditions under which the alert should be triggered. For metric alerts, the alerting
condition is specified using these fields in a nested structure value:
    * ```metric```: Specifies the unique ID of the metric to whose values these thresholds will be applied.
    * ```max```: (optional) the largest "ok" value the metric may take on; if missing or null, no maximum is enforced.
    * ```min```: (optional) the smallest "ok" value the metric may take on; if missing or null, no minimum is enforced.
    * Note that while both `min` and `max` are optional, at least one of them must be supplied to form a valid condition.

If successful, the response will contain the new alert definition that was created, including an inlined reference to the specified deployment resource, an indication of the notification channel(s) that will be used, and the generated `_id` value if applicable.



#### Update an existing alert definition

```
PUT https://telemetry-api.mongolab.com/loc/v0/alerts/:id
```


Update an alert definition by sending a **PUT** request with the entire alert definition in the body.  Accordingly, the JSON document in the request body is grammatically identical to the body described for the **POST** endpoint above.  However, two fields are handled specially during an update operation:

Any `_id` field in the request body is **ignored** in favor of the id that appears in the URL.  

The `deployment` field in the body is optional but, if given, must match the deployment information already present in the stored alert definition. That is, an update via **PUT** is not permitted to alter the deployment to which the alert is attached.


If successful, the response will contain the entire updated alert definition, including the proper `_id` field.  If the specified
alert definition does not exist, a 404 status will be returned.



#### Retrieve an alert definition

```
GET https://telemetry-api.mongolab.com/loc/v0/alerts/:id
```

Fetch the alert definition with the given `_id`.  If such an alert exists, it will be included in the response.  If not, a 404 status will be returned.


#### Retrieve a group of alert definitions

```
GET https://telemetry-api.mongolab.com/loc/v0/alerts?QUERY
```

Fetch a set of alert definitions based on zero or more constraining query parameters.
 
Optional query parameters:

* ```deployment=DEPLOYMENT_ID```: Find alerts that apply to the given deployment ID. To find alerts that do not specify an deployment, use ```deployment=NULL```.  If this parameter is
omitted, alerts for all deployments (permitted under acess control rules) will be returned.
* ```tags={"TAG_NAME":"TAG_VALUE", ...}```: Find alerts with the given tag(s).  If multiple tags are specified, only
alerts that have _all_ indicated tag values will be returned.


#### Delete an alert definition

```
DELETE https://telemetry-api.mongolab.com/loc/v0/alerts/:id
```

Delete the alert definition with the given `_id`, if such an alert exists. If not, a 404 status will be returned.