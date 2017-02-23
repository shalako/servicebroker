#Table of Contents#
  - [API Release Notes](#release-notes)
  - [Changes](#changes)
    - [Change Policy](#change-policy)
    - [Changes Since v2.10](#since-v2.10)
  - [API Overview](#api-overview)
  - [API Version Header](#version-header)
  - [Authentication](#authentication)
  - [Catalog Management](#catalog-mgmt)
    - [Adding a Broker to the Platform](#adding-broker-platform)
  - [Synchronous and Asynchronous Operations](#synchronous-asynchronous)
    - [Synchronous Operations](#synchronous-operations)
    - [Asynchronous Operations](#asynchronous-operations)
  - [Polling Last Operation](#polling)
    - [Polling Interval and Duration](#polling-interval-and-duration)
  - [Provisioning](#provisioning)
  - [Updating a Service Instance](#updating_service_instance)
  - [Binding](#binding)
    - [Types of Binding](#types-of-binding)
  - [Unbinding](#unbinding)
  - [Deprovisioning](#deprovisioning)
  - [Broker Errors](#broker-errors)
  - [Orphans](#orphans)

<div id="changes"/> 
## Changes ##

<div id="change-policy"/>
###Change Policy 

* Existing endpoints and fields will not be removed or renamed.
* New optional endpoints, or new HTTP methods for existing endpoints, may be
added to enable support for new features.
* New fields may be added to existing request/response messages.
These fields must be optional and should be ignored by clients and servers
that do not understand them.

<div id="since-v2.10"/>
##Changes Since v2.10 ##

* Add <tt>bindable</tt> field to [Plan Object](#PObject) to allow services to have both bindable and non-bindable plans.

<div id="api-overview"/> 
##API Overview 

The Service Broker API defines an HTTP interface between the services marketplace of a platform and service brokers.

The service broker is the component of the service that implements the Service Broker API, for which a platform's marketplace is a client. Service brokers are responsible for advertising a catalog of service offerings and service plans to the marketplace, and acting on requests from the marketplace for provisioning, binding, unbinding, and deprovisioning.

In general, provisioning reserves a resource on a service; we call this reserved resource a service instance. What a service instance represents can vary by service. Examples include a single database on a multi-tenant server, a dedicated cluster, or an account on a web application.

What a binding represents may also vary by service. In general creation of a binding either generates credentials necessary for accessing the resource or provides the service instance with information for a configuration change.

A platform marketplace may expose services from one or many service brokers, and an individual service broker may support one or many platform marketplaces using different URL prefixes and credentials.

<div id="version-header"/>  
## API Version Header

Requests from the platform to the service broker must contain a header that declares the version number of the Service Broker API that the marketplace will use:

`X-Broker-Api-Version: 2.11`

The version numbers are in the format `MAJOR.MINOR`, using semantic versioning such that 2.10 comes before 2.11.

This header allows brokers to reject requests from marketplaces for versions they do not support. While minor API revisions will always be additive, it is possible that brokers depend on a feature from a newer version of the API that is supported by the platform. In this scenario the broker may reject the request with `412 Precondition Failed` and provide a message that informs the operator of the required API version.

<div id="authentication"/> 
## Authentication

The marketplace must authenticate with the service broker using HTTP
basic authentication (the `Authorization:` header) on every request. The broker is responsible for validating the username and password and returning a `401 Unauthorized` message if credentials are invalid. It is recommended that brokers support secure communication from platform marketplaces over TLS.

<div id="catalog-mgmt"/>
## Catalog Management 

The first endpoint that a broker must implement is the service catalog.

The platform marketplace fetches this endpoint from all brokers in order to present an aggregated user-facing catalog.

Warnings for broker authors:

- Be cautious removing services and plans from their catalogs, as platform marketplaces may have provisioned service instances of these plans. Consider your deprecation strategy.
- Do not change the ids of services and plans. This action is likely to be evaluated by a platform marketplace as a removal of one plan and addition of another. See above warning about removal of plans.

The following sections describe catalog requests and responses in the Service Broker API.

### Request ###

#### Route ####
`GET /v2/catalog`

#### cURL ####
<pre class="terminal">
 $ curl -H "X-Broker-API-Version: 2.11" http://username:password@broker-url/v2/catalog
</pre>

### Response ###

| Status Code  | Description  |
|---|---|
| 200 OK  | The expected response body is below. |

#### Body - Schema of Service Objects ####

CLI and web clients have different needs with regard to service and plan names.
A CLI-friendly string is all lowercase, with no spaces.
Keep it short -- imagine a user having to type it as an argument for a longer
command.
A web-friendly display name is camel-cased with spaces and punctuation supported.

|  Response field | Type  | Description  |
|---|---|---|
|  services* |  array-of-service-objects |  Schema of service objects defined below. |

##### Service Objects #####

|  Response field |  Type | Description  |
|---|---|---|
|  name* | string  |  A CLI-friendly name of the service. All lowercase, no spaces. This must be globally unique within a platform marketplace. |
| id*  |  string | An identifier used to correlate this service in future requests to the broker. This must be globally unique within a platform marketplace. Using a GUID is recommended.  |
|  description* |  string | A short description of the service.  |
| tags  |  array-of-strings | Tags provide a flexible mechanism to expose a classification, attribute, or base technology of a service, enabling equivalent services to be swapped out without changes to dependent logic in applications, buildpacks, or other services. E.g. mysql, relational, redis, key-value, caching, messaging, amqp.  |
|  requires | array-of-strings  | A list of permissions that the user would have to give the service, if they provision it. The only permissions currently supported are <code>syslog\_drain</code>, <code>route\_forwarding</code> and <code>volume\_mount</code>.  |
| bindable*  | boolean  | Specifies whether instances of the service can be bound to applications. This specifies the default for all plans of this service. Plans can override this field (see [Plan Object](#PObject)).  |
| metadata  |  object | A list of metadata for a service offering.  |
| [dashboard_client](#DObject)  |  object |  Contains the data necessary to activate the Dashboard SSO feature for this service |
| plan\_updateable  | boolean  |  Whether the service supports upgrade/downgrade for some plans. Please note that the misspelling of the attribute <code>plan\_updatable</code> to <code>plan\_updateable</code> was done by mistake. We have opted to keep that misspelling instead of fixing it and thus breaking backward compatibility.  |
| [plans*](#PObject) | array-of-objects | A list of plans for this service, schema is defined below.|  |
| [schemas*](#SObject) | array-of-schemaset-objects | An ordered array of schemasets applicable for this service. The first schema-set matching the criteria applies. Schemasets should therefore be ordered from most specific to less specific. |

##### Dashboard Client Object <a name="DObject"></a> #####

| Response field  | Type  | Description  |
|---|---|---|
| id  | string  | The id of the Oauth client that the dashboard will use.  |
|  secret | string  | A secret for the dashboard client  |
|  redirect_uri |  string | A URI for the service dashboard. Validated by the OAuth token server when the dashboard requests a token.  |


##### Plan Object <a name="PObject"></a> #####

|  Response field | Type  | Description  |
|---|---|---|
| id*  | string  | An identifier used to correlate this plan in future requests to the broker. This must be globally unique within a platform marketplace. Using a GUID is recommended.  |
| name*  | string  | The CLI-friendly name of the plan. Must be unique within the service. All lowercase, no spaces.  |
| description*  | string  | A short description of the plan.  |
|  metadata | object  | A list of metadata for a service plan.  |
| free  | boolean  | When false, instances of this plan have a cost. The default is true  |
|  bindable | boolean  |  Specifies whether instances of the service plan can be bound to applications. This field is optional. If specified, this takes precedence over the <tt>bindable</tt> attribute of the service. If not specified, the default is derived from the service. |


##### Schema Object <a name="SObject"></a> #####

|  Response field | Type  | Description  |
|---|---|---|
| [applies-when*](#AWObject)  | object  | Defines the context in which a schema should apply through a list of selectors. The selectors should all match for the associated schemas to apply. There might be zero or more selectors. A context with no selectors always matches. |
| parameters | JSON schema object | The schema definitions for the input parameters. Schema definitions must be valid [JSON Schema draft v4](http://json-schema.org/). It is recommended for broker authors explicitly mention the JSON schema draft4 version, as this will prevent breaking changes when in the future the OSB spec starts supporting draft v6 JSON schema, with low readability impact. Each input parameter is expressed as property within a json object. Therefore, a service supporting no input parameters should result into a schema only validating an empty object `{}`. |


##### Applies-when Object <a name="AWObject"></a> #####

|  Response field | Type  | Description  |
|---|---|---|
| service-plan-id | guid  | Selects a service plan offering by its service plan id. If this field is missing then the schema will apply to all plans. |
| method | string | Selects the method to invoke on each object. Valid values are "create" and "update". If this field is missing then the schema will apply to all methods. |
| object | string | Selects the type of object on which schema should apply. Valid values are "service\_instance", "service\_binding".  If this field is missing then the schema will apply to all objects. |



\* Fields with an asterisk are required.

<pre>
{
  "services": [{
    "name": "fake-service",
    "id": "acb56d7c-XXXX-XXXX-XXXX-feb140a59a66",
    "description": "fake service",
    "tags": ["no-sql", "relational"],
    "requires": ["route_forwarding"],
    "max_db_per_node": 5,
    "bindable": true,
    "metadata": {
      "provider": {
        "name": "The name"
      },
      "listing": {
        "imageUrl": "http://example.com/cat.gif",
        "blurb": "Add a blurb here",
        "longDescription": "A long time ago, in a galaxy far far away..."
      },
      "displayName": "The Fake Broker"
    },
    "dashboard_client": {
      "id": "398e2f8e-XXXX-XXXX-XXXX-19a71ecbcf64",
      "secret": "277cabb0-XXXX-XXXX-XXXX-7822c0a90e5d",
      "redirect_uri": "http://localhost:1234"
    },
    "plan_updateable": true,
    "plans": [{
      "name": "fake-plan-1",
      "id": "d3031751-XXXX-XXXX-XXXX-a42377d3320e",
      "description": "Shared fake Server, 5tb persistent disk, 40 max concurrent connections",
      "max_storage_tb": 5,
      "metadata": {
        "costs":[
            {
               "amount":{
                  "usd":99.0
               },
               "unit":"MONTHLY"
            },
            {
               "amount":{
                  "usd":0.99
               },
               "unit":"1GB of messages over 20GB"
            }
         ],
        "bullets": [{
            "content": "Shared fake server"
          }, {
            "content": "5 TB storage"
          }, {
            "content": "40 concurrent connections"
        }]
      }
    }, {
      "name": "fake-plan-2",
      "id": "0f4008b5-XXXX-XXXX-XXXX-dace631cd648",
      "description": "Shared fake Server, 5tb persistent disk, 40 max concurrent connections. 100 async",
      "max_storage_tb": 5,
      "metadata": {
        "costs":[
            {
               "amount":{
                  "usd":199.0
               },
               "unit":"MONTHLY"
            },
            {
               "amount":{
                  "usd":0.99
               },
               "unit":"1GB of messages over 20GB"
            }
         ],
        "bullets": [{
          "content": "40 concurrent connections"
        }]
      }
    }]
  }],
  "schemas": [{
    "applies-when": {
      "method": "create",
      "object": "service_instance"
    },
    "parameters": {
      "$schema": "http://json-schema.org/draft-04/schema#",
      "type": "object",
      "properties": {
        "billing-account": {
          "description": "Billing account number used to charge use of shared fake server. Required in all plans",
          "type": "String"
        }
      }
    }
  }]
}
</pre>


<div id="adding-broker-platform"/>
### Adding a Broker to the Platform 

After implementing the first endpoint `GET /v2/catalog` documented [above](#catalog-mgmt), you must register the service broker with your platform to make your services and plans available to end users.

<div id="synchronous-asynchronous"</a>
## Synchronous and Asynchronous Operations 

Broker clients expect prompt responses to all API requests in order to provide users with fast feedback. Service broker authors should implement their brokers to respond promptly to all requests but must decide whether to implement synchronous or asynchronous responses. Brokers that can guarantee completion of the requested operation with the response should return the synchronous response. Brokers that cannot guarantee completion of the operation with the response should implement the asynchronous response.

Providing a synchronous response for a provision, update, or bind operation before actual completion causes confusion for users as their service may not be usable and they have no way to find out when it will be. Asynchronous responses set expectations for users that an operation is in progress and can also provide updates on the status of the operation.

Support for synchronous or asynchronous responses may vary by service offering, even by service plan.

<div id="synchronous-operations"/>
### Synchronous Operations 

To execute a request synchronously, the broker need only return the usual status codes: `201 CREATED` for provision and bind, and `200 OK` for update, unbind, and deprovision.

Brokers that support sychronous responses for provision, update, and delete can ignore the `accepts_incomplete=true` query parameter if it is provided by the client.

<div id="asynchronous-operations"/>
### Asynchronous Operations 

<p class='note'><strong>Note:</strong> Asynchronous operations are currently supported only for provision, update, and deprovision.</p>

For a broker to return an asynchronous response, the query parameter `accepts_incomplete=true` must be included the request. If the parameter is not included or is set to `false`, and the broker cannot fulfill the request synchronously (guaranteeing that the operation is complete on response), then the broker should reject the request with the status code `422 UNPROCESSABLE ENTITY` and the following body:

<pre class="terminal">
{
  "error": "AsyncRequired",
  "description": "This service plan requires client support for asynchronous service operations."
}
</pre>

If the query parameter described above is present, and the broker executes the request asynchronously, the broker must return the asynchronous response `202 ACCEPTED`. The response body should be the same as if the broker were serving the request synchronously.

An asynchronous response triggers the platform marketplace to poll the endpoint `GET /v2/service_instances/:guid/last_operation` until the broker indicates that the requested operation has succeeded or failed. Brokers may include a status message with each response for the `last_operation` endpoint that provides visibility to end users as to the progress of the operation.

#### Blocking Operations ####

The marketplace must ensure that service brokers do not receive requests for an instance while an asynchronous operation is in progress. For example, if a broker is in the process of provisioning an instance asynchronously, the marketplace must not allow any update, bind, unbind, or deprovision requests to be made through the platform. A user who attempts to perform one of these actions while an operation is already in progress must receive an HTTP 400 response with the error message: `Another operation for this service instance is in progress`.

<div id="polling"/>
## Polling Last Operation 

When a broker returns status code `202 ACCEPTED` for [provision](#provisioning), [update](#updating_service_instance), or [deprovision](#deprovisioning), the platform will begin polling the `/v2/service_instances/:guid/last_operation` endpoint to obtain the state of the last requested operation. The broker response must contain the field `state` and an optional field `description`.

Valid values for `state` are `in progress`, `succeeded`, and `failed`. The platform will poll the `last_operation` endpoint as long as the broker returns `"state": "in progress"`. Returning `"state": "succeeded"` or `"state": "failed"` will cause the platform to cease polling. The value provided for `description` will be passed through to the platform API client and can be used to provide additional detail for users about the progress of the operation.

### Request ###

##### Route #####
`GET /v2/service_instances/:instance_id/last_operation`

##### Parameters #####

The request provides these query string parameters as useful hints for brokers.

|  Query-String Field | Type  | Description  |
|---|---|---|
| service_id  |  string | ID of the service from the catalog.  |
| plan_id  | string  | ID of the plan from the catalog.  |
| operation  |  string | A broker-provided identifier for the operation. When a value for <code>operation</code> is included with asynchronous responses for [Provision](#provisioning), [Update](#updating_service_instance), and [Deprovision](#deprovisioning) requests, the broker client should provide the same value using this query parameter as a URL-encoded string.  |

<p class="note"><strong>Note:</strong> Although the request query parameters <code>service_id</code> and <code>plan_id</code> are not required, the platform should include them on all <code>last_operation</code> requests it makes to service brokers.</p>

##### cURL #####
<pre class="terminal">
$ curl http://username:password@broker-url/v2/service_instances/:instance_id/last_operation
</pre>

### Response ###

| Status Code  |  Description |
|---|---|
| 200 OK |  The expected response body is below. |
|  410 GONE | Appropriate only for asynchronous delete operations. The platform should consider this response a success and remove the resource from its database. The expected response body is <code>{}</code>. Returning this while the platform is polling for create or update operations should be interpreted as an invalid response and the platform should continue polling.  |

Responses with any other status code should be interpreted as an error or invalid response. The platform should continue polling until the broker returns a valid response or the [maximum polling duration](#polling-interval-and-duration) is reached. Brokers may use the `description` field to expose user-facing error messages about the operation state; for more info see [Broker Errors](#broker-errors).

##### Body #####

All response bodies must be a valid JSON Object (`{}`). This is for future compatibility; it will be easier to add fields in the future if JSON is expected rather than to support the cases when a JSON body may or may not be returned.

For success responses, the following fields are valid.

|  Response field | Type  | Description  |
|---|---|---|
| state*  | string  | Valid values are <code>in progress</code>, <code>succeeded</code>, and <code>failed</code>. While <code>"state": "in progress"</code>, the platform should continue polling. A response with <code>"state": "succeeded"</code> or <code>"state": "failed"</code> should cause the platform to cease polling.  |
|  description |  string | Optional field. A user-facing message displayed to the platform API client. Can be used to tell the user details about the status of the operation.  |

\* Fields with an asterisk are required.

<pre class="terminal">
{
  "state": "in progress",
  "description": "Creating service (10% complete)."
}
</pre>

<div id="polling-interval-and-duration"/>
### Polling Interval and Duration    

The frequency and maximum duration of polling may vary by platform client. If a platform has a max polling duration and this limit is reached, the platform will cease polling and the operation state will be considered `failed`.

<div id="provisioning"/> 
## Provisioning

When the broker receives a provision request from the platform, it should take whatever action is necessary to create a new resource. What provisioning represents varies by service and plan, although there are several common use cases. For a MySQL service, provisioning could result in an empty dedicated database server running on its own VM or an empty schema on a shared database server. For non-data services, provisioning could just mean an account on an multi-tenant SaaS application.

### Request ###

##### Route #####
`PUT /v2/service_instances/:instance_id`

The `:instance_id` of a service instance is provided by the platform. This ID will be used for future requests (bind and deprovision), so the broker must use it to correlate the resource it creates.

##### Body #####
| Request field  | Type  |  Description |
|---|---|---|
| service_id*  | string  | The ID of the service (from the catalog). Must be globally unique.  |
| plan_id*  | string  | The ID of the plan (from the catalog) for which the service instance has been requested. Must be unique to a service.  |
|  parameters |  JSON object | Configuration options for the service instance.  |
|  accepts_incomplete | boolean  | A value of true indicates that the marketplace and its clients support asynchronous broker operations. If this parameter is not included in the request, and the broker can only provision an instance of the requested plan asynchronously, the broker should reject the request with a 422 as described below.  |
| organization_guid*  | string  | The platform GUID for the organization under which the service is to be provisioned. Although most brokers will not use this field, it may be helpful for executing operations on a user's behalf.  |
|  space_guid* |  string |  The identifier for the project space within the platform organization. Although most brokers will not use this field, it may be helpful for executing operations on a user's behalf. |

\* Fields with an asterisk are required.

<pre class="terminal">
{
  "service_id": "service-guid-here",
  "plan_id": "plan-guid-here",
  "organization_guid": "org-guid-here",
  "space_guid": "space-guid-here",
  "parameters": {
    "parameter1": 1,
    "parameter2": "foo"
  }
}
</pre>

##### cURL #####
<pre class="terminal">
$ curl http://username:password@broker-url/v2/service_instances/:instance_id -d '{
  "service_id": "service-guid-here",
  "plan_id": "plan-guid-here",
  "organization_guid": "org-guid-here",
  "space_guid": "space-guid-here",
  "parameters": {
    "parameter1": 1,
    "parameter2": "foo"
  }
}' -X PUT -H "X-Broker-API-Version: 2.11" -H "Content-Type: application/json"
</pre>

### Response ###

| Status Code  | Description  |
|---|---|
| 201 Created  | Service instance has been provisioned. The expected response body is below.  |
| 200 OK  |  May be returned if the service instance already exists and the requested parameters are identical to the existing service instance. The expected response body is below. |
| 202 Accepted  | Service instance provisioning is in progress. This triggers the platform marketplace to poll the [Service Instance Last Operation Endpoint](#polling) for operation status.  |
| 409 Conflict  | Should be returned if a service instance with the same id already exists but with different attributes. The expected response body is <code>{}</code>.  |
| 422 Unprocessable Entity  | Should be returned if the broker only supports asynchronous provisioning for the requested plan and the request did not include <code>?accepts_incomplete=true</code>. The expected response body is: <code>{ "error": "AsyncRequired", "description": "This service plan requires client support for asynchronous service operations." }</code>, as described below.  |


Responses with any other status code will be interpreted as a failure. Brokers can include a user-facing message in the `description` field; for details see [Broker Errors](#broker-errors).

##### Body #####

All response bodies must be a valid JSON Object (`{}`). This is for future compatibility; it will be easier to add fields in the future if JSON is expected rather than to support the cases when a JSON body may or may not be returned.

For success responses, a broker may return the following fields. For error responses, see [Broker Errors](#broker-errors).

| Response field  |  Type | Description  |
|---|---|---|
|  dashboard_url | string  |  The URL of a web-based management user interface for the service instance; we refer to this as a service dashboard. The URL should contain enough information for the dashboard to identify the resource being accessed (<code>9189kdfsk0vfnku</code> in the example below). |
|  operation |  string | For asynchronous responses, service brokers may return an identifier representing the operation. The value of this field should be provided by the broker client with requests to the [Last Operation](#polling) endpoint in a URL encoded query parameter.  |

\* Fields with an asterisk are required.

<pre class="terminal">
{
 "dashboard\_url": "<span>http</span>://example-dashboard.example.com/9189kdfsk0vfnku",
 "operation": "task\_10"
}
</pre>

<div id="updating_service_instance"/>
## Updating a Service Instance 

By implementing this endpoint, service broker authors can enable users to modify two attributes of an existing service instance: the service plan and parameters. By changing the service plan, users can upgrade or downgrade their service instance to other plans. By modifying properties, users can change configuration options that are specific to a service or plan.

To enable support for the update of the plan, a broker should declare support per service by including `plan_updateable: true` in its [catalog endpoint](#catalog-mgmt).

Not all permutations of plan changes are expected to be supported. For example, a service may support upgrading from plan "shared small" to "shared large" but not to plan "dedicated". It is up to the broker to validate whether a particular permutation of plan change is supported. If a particular plan change is not supported, the broker should return a meaningful error message in response.

### Request ###

##### Route #####
`PATCH /v2/service_instances/:instance_id`

`:instance_id` is the global unique ID of a previously-provisioned service instance.

##### Body #####

| Request Field  | Type  |  Description |
|---|---|---|
| service\_id*  | string  | The ID of the service (from the catalog). Must be globally unique.  |
| plan\_id  | string  | The ID of the plan (from the catalog) for which the service instance has been requested. Must be unique to a service.  |
| parameters  | JSON object  | Configuration options for the service instance.  |
| accepts\_incomplete  |  boolean | A value of true indicates that the marketplace and its clients support asynchronous broker operations. If this parameter is not included in the request, and the broker can only provision an instance of the requested plan asynchronously, the broker should reject the request with a 422 as described below.  |
| previous\_values  | object  |  Information about the instance prior to the update. |
| previous\_values.service_id  | string  | ID of the service for the instance.  |
| previous\_values.plan_id  |  string | ID of the plan prior to the update.  |
| previous\_values.organization_id  | string  | ID of the organization specified for the instance.  |
| previous\_values.space_id  | string  | ID of the space specified for the instance.  |

\* Fields with an asterisk are required.

<pre class="terminal">
{
  "service_id": "service-guid-here",
  "plan_id": "plan-guid-here",
  "parameters": {
    "parameter1": 1,
    "parameter2": "foo"
  },
  "previous_values": {
    "plan_id": "old-plan-guid-here",
    "service_id": "service-guid-here",
    "organization_id": "org-guid-here",
    "space_id": "space-guid-here"
  }
}
</pre>

##### cURL #####
<pre class="terminal">
$ curl http://username:password@broker-url/v2/service_instances/:instance_id -d '{
  "service_id": "service-guid-here",
  "plan_id": "plan-guid-here",
  "parameters": {
    "parameter1": 1,
    "parameter2": "foo"
  },
  "previous_values": {
    "plan_id": "old-plan-guid-here",
    "service_id": "service-guid-here",
    "organization_id": "org-guid-here",
    "space_id": "space-guid-here"
  }
}' -X PATCH -H "X-Broker-API-Version: 2.11" -H "Content-Type: application/json"
</pre>

### Response ###

| Status Code  | Description  |
|---|---|
| 200 OK  | The requests changes have been applied. The expected response body is <code>{}</code>.  |
| 202 Accepted  |  Service instance update is in progress. This triggers the platform marketplace to poll the [Service Instance Last Operation Endpoint](#polling) for operation status. |
| 422 Unprocessable entity  | May be returned if the requested change is not supported or if the request cannot currently be fulfilled due to the state of the instance (e.g. instance utilization is over the quota of the requested plan). Broker should include a user-facing message in the body; for details see [Broker Errors](#broker-errors).  Additionally, a 422 can also be returned if the broker only supports asynchronous update for the requested plan and the request did not include <code>?accepts_incomplete=true</code>; in this case the expected response body is: <code>{ "error": "AsyncRequired", "description": "This service plan requires client support for asynchronous service operations." }</code>  |

Responses with any other status code will be interpreted as a failure. Brokers can include a user-facing message in the `description` field; for details see [Broker Errors](#broker-errors).

##### Body #####

All response bodies must be a valid JSON Object (`{}`). This is for future compatibility; it will be easier to add fields in the future if JSON is expected rather than to support the cases when a JSON body may or may not be returned.

For success responses, a broker may return the following field. Others will be ignored. For error responses, see [Broker Errors](#broker-errors).

| Response field  | Type  | Description  |
|---|---|---|
| operation  | string  |  For asynchronous responses, service brokers may return an identifier representing the operation. The value of this field should be provided by the broker client with requests to the [Last Operation](#polling) endpoint in a URL encoded query parameter. |

\* Fields with an asterisk are required.

<pre class="terminal">
{
 "operation": "task_10"
}
</pre>


<div id="binding"/>
## Binding 

If `bindable:true` is declared for a service or plan in the [Catalog](#catalog-mgmt) endpoint, broker clients may request generation of a service binding. 

<p class="note"><strong>Note</strong>: Not all services must be bindable --- some deliver value just from being provisioned. Brokers that offer services that are bindable should declare them as such using <code>bindable: true</code> in the <a href="#catalog-mgmt">Catalog</a>. Brokers that do not offer any bindable services do not need to implement the endpoint for bind requests.</p>

<div id="types-of-binding"/>
### Types of Binding 

#### Credentials ####

Credentials are a set of information used by an application or a user to utilize the service instance. If the broker supports generation of credentials it should return `credentials` in the response for a request to create a service binding. Credentials should be unique whenever possible, so access can be revoked for each binding without affecting consumers of other bindings for the service instance. 

#### Log Drain ####

There are a class of service offerings that provide aggregation, indexing, and analysis of log data. To utilize these services an application that generates logs needs information for the location to which it should stream logs. If a broker represents one of these services, it may optionally return a `syslog_drain_url` in the response for a request to create a service binding, to which logs may be streamed.  

The `requires` field in the [Catalog](#catalog-mgmt) endpoint enables a platform marketplace to validate a response for create binding that includes a `syslog_drain_url`. Platform marketplaces should consider a broker's response invalid if it includes a `syslog_drain_url` and `"requires":["syslog_drain"]` is not present in the [Catalog](#catalog-mgmt) endpoint.

<div id= "route_services"/>
#### Route Services 

There are a class of service offerings that intermediate requests to applications, performing functions such as rate limiting or authorization. To configure a service instance with behavior specific to an application's routable address, a broker client may send the address along with the request to create a binding using `"bind_resource":{"route":"some-address.com"}`.

Some platforms may support proxying of application requests to service instances. In this case the platform needs to know where to send application requests; to facilitate this, the broker may return a `route_service_url` in the response for a request to create a binding. Not all services of this type expect to receive requests proxied by the platform; some services will have been configured out-of-band to intermediate requests to applications. In this case, the broker will not return `route_service_url` in response to the create binding request. By sending `bind-resource` as described above, the platform enables dynamic configuration of a service instance already in the application request path for the route, requiring no change in the platform routing tier. 

The `requires` field in the [Catalog](#catalog-mgmt) endpoint enables a platform marketplace to validate requests to create bindings. A platform may opt to reject requests to create bindings when a broker has declared `"requires":["route_forwarding"]` for a service in the catalog endpoint.

#### Volume Services ####

There are a class of services that provide network storage to applications via volume mounts in the application container. A service broker may return data required for this configuration with `volume_mount` in response to the request to create a binding. 

The `requires` field in the [Catalog](#catalog-mgmt) endpoint enables a platform marketplace to validate a response for create binding that includes a `volume_mounts`. Platform marketplaces should consider a broker's response invalid if it includes a `volume_mounts` and `"requires":["volume_mount"]` is not present in the [Catalog](#catalog-mgmt) endpoint.

### Request ###

##### Route #####
`PUT /v2/service_instances/:instance_id/service_bindings/:binding_id`

The `:instance_id` is the ID of a previously-provisioned service instance. The `:binding_id` is also provided by the platform. This ID will be used for future unbind requests, so the broker must use it to correlate
the resource it creates.

##### Body #####

| Request Field  | Type  | Description  |
|---|---|---|
| service_id*  | string  | ID of the service from the catalog.  |
| plan_id*  | string  | ID of the plan from the catalog.  |
| app_guid  | string  | Deprecated in favor of <code>bind\_resource.app\_guid</code>. GUID of an application associated with the binding to be created.  |
| bind_resource  | JSON object  | A JSON object that contains data for platform resources associated with the binding to be created. Current valid values include <code>app\_guid</code> for [credentials](#types-of-binding) and <code>route</code> for [route services](#route_services).  |
| parameters | JSON object  |  Configuration options for the service binding. |   |

\* Fields with an asterisk are required.

<pre class="terminal">
{
  "service_id": "service-guid-here",
  "plan_id": "plan-guid-here",
  "bind_resource": {
    "app_guid": "app-guid-here"
  },
  "parameters": {
    "parameter1-name-here": 1,
    "parameter2-name-here": "parameter2-value-here"
  }
}
</pre>


##### cURL #####
<pre class="terminal">
$ curl http://username:password@broker-url/v2/service_instances/:instance_id/service_bindings/:binding_id -d '{
  "service_id": "service-guid-here",
  "plan_id": "plan-guid-here",
  "bind_resource": {
    "app_guid": "app-guid-here"
  },
  "parameters": {
    "parameter1-name-here": 1,
    "parameter2-name-here": "parameter2-value-here"
  }
}' -X PUT
</pre>

### Response ###

| Status Code  | Description  |
|---|---|
| 201 Created  |  Binding has been created. The expected response body is below. |
| 200 OK  |  May be returned if the binding already exists and the requested parameters are identical to the existing binding. The expected response body is below. |
| 409 Conflict  | Should be returned if the requested binding already exists. The expected response body is <code>{}</code>, though the description field can be used to return a user-facing error message, as described in [Broker Errors](#broker-errors).  |
| 422 Unprocessable Entity  | Should be returned if the broker requires that <code>app_guid</code> be included in the request body. The expected response body is: <code>{ "error": "RequiresApp", "description": "This service supports generation of credentials through binding an application only." }</code>  |

Responses with any other status code will be interpreted as a failure and an unbind request will be sent to the broker to prevent an orphan being created on the broker. Brokers can include a user-facing message in the `description` field; for details see [Broker Errors](#broker-errors).

##### Body #####

All response bodies must be a valid JSON Object (`{}`). This is for future compatibility; it will be easier to add fields in the future if JSON is expected rather than to support the cases when a JSON body may or may not be returned.

For success responses, the following fields are supported. Others will be ignored. For error responses, see [Broker Errors](#broker-errors).

|  Response Field | Type  | Description  |
|---|---|---|
| credentials  | object  |  A free-form hash of credentials that may be used by applications or users to access the service. |
| syslog\_drain_url  | string  | A URL to which logs may be streamed. <code>"requires":["syslog\_drain"]</code> must be declared in the [Catalog](#catalog-mgmt) endpoint or the platform should consider the response invalid.  |
| route\_service_url  | string  |  A URL to which the platform may proxy requests for the address sent with <code>bind\_resource.route</code> in the request body. <code>"requires":["route\_forwarding"]</code> must be declared in the [Catalog](#catalog-mgmt) endpoint or the platform should consider the response invalid. |
| volume\_mounts  | array-of-objects  | An array of configuration for mounting volumes. <code>"requires":["volume_mount"]</code> must be declared in the [Catalog](#catalog-mgmt) endpoint or the platform should consider the response invalid.  |

<pre class="terminal">
    {
      "credentials": {
        "uri": "mysql://mysqluser:pass@mysqlhost:3306/dbname",
        "username": "mysqluser",
        "password": "pass",
        "host": "mysqlhost",
        "port": 3306,
        "database": "dbname"
      }
    }
</pre>

<div id="unbinding"/>
## Unbinding 

<p class="note"><strong>Note</strong>: Brokers that do not provide any bindable services or plans do not need to implement this endpoint.</p>

When a broker receives an unbind request from the marketplace, it should delete any resources associated with the binding. In the case where credentials were generated, this may result in requests to the service instance failing to authenticate.

### Request ###

##### Route #####
`DELETE /v2/service_instances/:instance_id/service_bindings/:binding_id`

The `:instance_id` is the ID of a previously-provisioned service instance. The `:binding_id` is the ID of a previously provisioned binding for that instance.

##### Parameters #####

The request provides these query string parameters as useful hints for brokers.

| Query-String Field  | Type  | Description |
|---|---|---|
| service_id*  | string | ID of the service from the catalog. |
| plan_id*  | string  | ID of the plan from the catalog. |

\* Query parameters with an asterisk are required.

##### cURL #####
<pre class="terminal">
$ curl 'http://username:password@broker-url/v2/service_instances/:instance_id/
  service_bindings/:binding_id?service_id=service-id-here&plan_id=plan-id-here' -X DELETE -H "X-Broker-API-Version: 2.11"
</pre>

### Response ###

| Status Code  | Description  |
|---|---|
| 200 OK  | Binding was deleted. The expected response body is <code>{}</code>.  |
| 410 Gone  | Should be returned if the binding does not exist. The expected response body is <code>{}</code>.  |

Responses with any other status code will be interpreted as a failure and the binding will remain in the marketplace database. Brokers can include a user-facing message in the `description` field; for details see [Broker Errors](#broker-errors).

##### Body #####

All response bodies must be a valid JSON Object (`{}`). This is for future compatibility; it will be easier to add fields in the future if JSON is expected rather than to support the cases when a JSON body may or may not be returned.

For a success response, the expected response body is `{}`.

<div id="deprovisioning"/>
## Deprovisioning 

When a broker receives a deprovision request from the marketplace, it should
delete any resources it created during the provision.
Usually this means that all resources are immediately reclaimed for future
provisions.

### Request ###

##### Route #####
`DELETE /v2/service_instances/:instance_id`

`:instance_id` is the identifier of a previously provisioned instance.

##### Parameters #####

The request provides these query string parameters as useful hints for brokers.

| Query-String Field | Type | Description |
|---|---|---|
| service_id* | string | ID of the service from the catalog. |
| plan_id* | string | ID of the plan from the catalog. |
| accepts_incomplete  | boolean | A value of true indicates that both the marketplace and the requesting client support asynchronous deprovisioning. If this parameter is not included in the request, and the broker can only deprovision an instance of the requested plan asynchronously, the broker should reject the request with a 422 as described below. |

\* Query parameters with an asterisk are required.

##### cURL #####
<pre class="terminal">
$ curl 'http://username:password@broker-url/v2/service_instances/:instance_id?service_id=
    service-id-here&plan_id=plan-id-here' -X DELETE -H "X-Broker-API-Version: 2.11"
</pre>

### Response ###

| Status Code | Description |
|---|---|
| 200 OK | Service instance was deleted. The expected response body is <code>{}</code>. |
| 202 Accepted | Service instance deletion is in progress. This triggers the marketplace to poll the [Service Instance Last Operation Endpoint](#polling) for operation status.  |
| 410 Gone | Should be returned if the service instance does not exist. The expected response body is <code>{}</code>.  |
| 422 Unprocessable Entity | Should be returned if the broker only supports asynchronous deprovisioning for the requested plan and the request did not include <code>?accepts_incomplete=true</code>. The expected response body is: <code>{ "error": "AsyncRequired", "description": "This service plan requires client support for asynchronous service operations." }</code>, as described below.  |

Responses with any other status code will be interpreted as a failure and the service instance will remain in the marketplace database. Brokers can include a user-facing message in the `description` field; for details see [Broker Errors](#broker-errors).

##### Body #####

All response bodies must be a valid JSON Object (`{}`). This is for future compatibility; it will be easier to add fields in the future if JSON is expected rather than to support the cases when a JSON body may or may not be returned.

For success responses, the following fields are supported. Others will be ignored. For error responses, see [Broker Errors](#broker-errors).


|  Response field | Type | Description |
|---|---|---|
| operation  | string  | For asynchronous responses, service brokers may return an identifier representing the operation. The value of this field should be provided by the broker client with requests to the [Last Operation](#polling) endpoint in a URL encoded query parameter. |

\* Fields with an asterisk are required.

<pre class="terminal">
{
 "operation": "task_10"
}
</pre>

<div id="broker-errors"/>
## Broker Errors 

### Response ###

Broker failures beyond the scope of the well-defined HTTP response codes listed
above (like 410 on delete) should return an appropriate HTTP response code
(chosen to accurately reflect the nature of the failure) and a body containing a valid JSON Object (not an array).

##### Body #####

All response bodies must be a valid JSON Object (`{}`). This is for future compatibility; it will be easier to add fields in the future if JSON is expected rather than to support the cases when a JSON body may or may not be returned.

For error responses, the following fields are valid. Others will be ignored. If an empty JSON object is returned in the body `{}`, a generic message containing the HTTP response code returned by the broker will be displayed to the requester.

| Response Field | Type | Description |
|---|---|---|
| description | string |  A meaningful error message explaining why the request failed. |

<pre class="terminal">
{
  "description": "Your account has exceeded its quota for service instances. Please contact support at http://support.example.com."
}
</pre>

<div id="orphans"/> 
## Orphans

The platform marketplace is the source of truth for service instances and bindings. Service brokers are expected to have successfully provisioned all the instances and bindings that the marketplace knows about, and none that it doesn't.

Orphans can result if the broker does not return a response before a request from the marketplace times out (typically 60 seconds). For example, if a broker does not return a response to a provision request before the request times out, the broker might eventually succeed in provisioning an instance after the marketplace considers the request a failure. This results in an orphan instance on the broker's side.

To mitigate orphan instances and bindings, the marketplace should attempt to delete resources it cannot be sure were successfully created, and should keep trying to delete them until the broker responds with a success. 

Platforms should initiate orphan mitigation in the following scenarios:

| Status code of broker response | Platform interpretation of response | Platform initiates orphan mitigation? |
|---|---|---|
| 200 | Success | No |
| 200 with malformed response |  Failure | No |
| 201 | Success | No |
| 201 with malformed response | Failure | Yes |
| All other 2xx | Failure | Yes |
| 408 | Failure due to timeout | Yes |
| All other 4xx | Broker rejected request | No |
| 5xx | Broker error | Yes |
| Timeout | Failure | Yes |

If the platform marketplace encounters an internal error provisioning an instance or binding (for example, saving to the database fails), then it should at least send a single delete or unbind request to the service broker to prevent creation of an orphan.
