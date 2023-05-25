---
layout: default
title: IP2geo processor
grand_parent: Ingest APIs
parent: Processors
nav_order: 20
---

# IP2geo processor 
Introduced 2.9
{: .label .label-purple }

The OpenSearch `IP2geo` processor adds information about the geographical location of an IPv4 or IPv6 address. The `IP2geo` processor uses IP geolocation (GeoIP) data from an external endpoint and therefore requires an additional component, `datasource`, that defines from where to download GeoIP data and how frequently to update the data.

## Installing the geospatial plugin

Before getting started with using the `IP2geo` processor, the `opensearch-geospatial` plugin must be installed. Learn more in the [Installing plugins]({{site.url}}{{site.baseurl}}/install-and-configure/plugins/) section.

## Creating the IP2geo data source

OpenSearch provides the following endpoints for GeoLite2 City, GeoLite2 Country, and GeoLite2 ASN databases from [MaxMind](http://dev.maxmind.com/geoip/geoip2/geolite2/), shared under the CC BY-SA 4.0 license:

* GeoLite2 City: https://geoip.maps.opensearch.org/v1/geolite2-city/manifest.json
* GeoLite2 Country: https://geoip.maps.opensearch.org/v1/geolite2-country/manifest.json
* GeoLite2 ASN: https://geoip.maps.opensearch.org/v1/geolite2-asn/manifest.json

If OpenSearch cannot update a data source from those endpoints in 30 days, OpenSearch does not add GeoIP data to documents, instead it adds `"error":"ip2geo_data_expired"`.

The following table lists the IP2geo data source options.

| Name | Required | Default | Description |
|------|----------|---------|-------------|
| endpoint | no | https://geoip.maps.opensearch.org/v1/geolite2-city/manifest.json | The endpoint for downloading the GeoIP data|
| update_interval_in_days | no | 3 | The frequency in days for updating the GeoIP data; minimum value is 1. |

The following code example shows how to create an IP2Geo data source.

#### Example: PUT Request

````json
PUT /_plugins/geospatial/ip2geo/datasource/my-datasource
{
    "endpoint" : "https://geoip.maps.opensearch.org/v1/geolite2-city/manifest.json",
    "update_interval_in_days" : 3
}
```

The following code example shows the reponse to the preceding request. A true response means the request was successful and the server was able to process the request. A false reponse means check the request to make sure it is valid, check the URL to make sure it is correct, or try again.

#### Example: Response

```json
{
    "acknowledged":true
}
```

## Sending a GET request

To get information about an IP2geo data source, send a GET request.  

#### Example: GET Request

```
GET /_plugins/geospatial/ip2geo/datasource/my-datasource
```

#### Example: Response

```json
{
  "datasources": [
    {
      "name": "my-datasource",
      "state": "AVAILABLE",
      "endpoint": "https://geoip.maps.opensearch.org/v1/geolite2-city/manifest.json",
      "update_interval_in_days": 3,
      "next_update_at_in_epoch_millis": 1685125612373,
      "database": {
        "provider": "maxmind",
        "sha256_hash": "0SmTZgtTRjWa5lXR+XFCqrZcT495jL5XUcJlpMj0uEA=",
        "updated_at_in_epoch_millis": 1684429230000,
        "valid_for_in_days": 30,
        "fields": [
          "country_iso_code",
          "country_name",
          "continent_name",
          "region_iso_code",
          "region_name",
          "city_name",
          "time_zone",
          "location"
        ]
      },
      "update_stats": {
        "last_succeeded_at_in_epoch_millis": 1684866730192,
        "last_processing_time_in_millis": 317640,
        "last_failed_at_in_epoch_millis": 1684866730492,
        "last_skipped_at_in_epoch_millis": 1684866730292
      }
    }
  ]
}
```

## Updating an IP2geo data source

To update succesfully an IP2geo data source, the GeoIP database from the latest-released database's endpoint must contain the same fields as those in the currently used database. Otherwise, the update fails. 

#### Example: Update request

```json
{
    "endpoint": https://geoip.maps.opensearch.org/v1/geolite2-city/manifest.json,
    "update_interval_in_days": 10
}
```

#### Example: Response

```json
{
    "acknowledged":true
}
```

## Deleting an IP2geo data source

 To delete an IP2geo data source, you first must delete all processors associated with the data source. Otherwise, if you have other processors that use the data source, the DELETE request fails. 

#### Example: DELETE request

```
DELETE /_plugins/geospatial/ip2geo/datasource/my-datasource
```

#### Example: Response
{
  "acknowledged": true
}


## Creating the IP2geo processor

Once the IP2geo data source is created, you can create the processor. 

#### Example: Create processor request

```json
{
   "description":"convert ip to geo",
   "processors":[
    {
        "ip2geo":{
            "field":"ip",
            "datasource":"my-datasource"
        }
    }
   ] 
}

#### Example: Response

```json
{
	"acknowledged": true
}
```

## Using the IP2geo processor in a pipeline

The following table lists `ip2geo` field options:

| Name | Required | Default | Description |
|------|----------|---------|-------------|
| field | yes | - | The field to get the ip address for the geographical lookup. |
| datasource | yes | - | The data source name to look up geographical information. |
| properties | no |  All fields in `datasource`. | The field that controls what properties are added to `target_field` based on the GeoIP lookup. |
| target_field | no | ip2geo | The field that holds the geographical information looked up from the data source. |
| database | no | geoip.maps.opensearch.org/v1/geolite2-city/manifest.json | The database filename referring to a database the module ships with or a custom database in the ingest-geoip config directory. |
| ignore_missing | no | false | If `true` and `field` does not exist, the processor quietly exits without modifying the document. |
| first_only | no | true | If `true` only first found geoip data is returned, even if field contains an array. |

The following code is an example of using the `IP2geo` processor to add the geographical information to the `ip2geo` field based on the `ip` field.

```json
PUT /_ingest/pipeline/ip2geo
{
   "description":"convert ip to geo",
   "processors":[
    {
        "ip2geo":{
            "field":"ip",
            "datasource":"my-datasource"
        }
    }
   ] 
}

PUT /my-index/_doc/my-id?pipeline=ip2geo
{
  "ip": "172.0.0.1"
}

GET /my-index/_doc/my-id
Which returns:

{
   "_index":"my-index",
   "_id":"my-id",
   "_version":1,
   "_seq_no":0,
   "_primary_term":1,
   "found":true,
   "_source":{
      "my_ip_field":"172.0.0.1",
      "ip2geo":{
         "continent_name":"North America",
         "region_iso_code":"AL",
         "city_name":"Calera",
         "country_iso_code":"US",
         "country_name":"United States",
         "region_name":"Alabama",
         "location":"33.1063,-86.7583",
         "time_zone":"America/Chicago"
      }
   }
}
```

## Node settings

The IP2geo data source and `IP2geo` processor have the following node settings.


| IP2geo Data source | Description | Setting |
|--------------------|-------------|---------|
| plugins.geospatial.ip2geo.datasource.endpoint | Default endpoint for creating the data source API. | Defaults to https://geoip.maps.opensearch.org/v1/geolite2-city/manifest.json. |
| plugins.geospatial.ip2geo.datasource.update_interval_in_days | Default update interval for creating the data source API. | Defaults to 3. |
| plugins.geospatial.ip2geo.datasource.indexing_bulk_size | Maximum number of documents to ingest in a bulk request during the IP2geo data source creation process. | Defaults to 10,000. |

 IP2geo processor | Description | Setting |
|--------------------|-------------|---------|
| plugins.geospatial.ip2geo.processor.max_bundle_size | Maximum number of searches to include in a multi-search request when enriching documents. | Defaults to 100.
| plugins.geospatial.ip2geo.processor.max_concurrent_searches | Maximum number of concurrent multi-search requests to run when enriching documents. | The default depends on your node count and search thread pool size. Higher values can improve performance, but risk overloading the cluster.