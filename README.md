# Sample Elastic Kibana dashboards for CICS Performance Analyzer

This repository demonstrates visualizing json lines data from IBM CICS Performance Analyzer in Elastic Kibana dashboards.
Check CICS SupportPac "[CA10:CICS Performance Analyzer for z/OS - Output to JSON Lines](https://www.ibm.com/support/pages/node/1282648)" for more information.


You can use this repository to:

-   Configure your own Elastic Stack instance with the sample Kibana dashboards and sample data
-   Visualize your own data in the sample dashboards

## Repository contents

-   `elasticsearch/`\
    Contains an Elasticsearch index template (component, not legacy) and lifecycle policy.

-   `kibana/`\
    Contains Kibana saved objects, including dashboards and related objects such as index patterns. Also included is a transform template. 

-   `logstash/`\
    Contains a Logstash pipeline configuration.

## Installing the dashboards in an existing Elastic Stack instance

The following instructions assume that you have an existing Elastic Stack instance into which you want to install the Kibana dashboards that are provided by this repository. For details on installing Elastic Stack, see the documentation on the Elastic website.

### Create an Elasticsearch index lifecycle management (ILM) policy

Use the `samples/elasticsearch/cicspa-ilm-policy.json` file as the body of an Elasticsearch create lifecycle policy API request.

Example API request:

````sh
curl -X PUT -H "Content-Type: application/json" -d @cicspa-ilm-policy.json "http://localhost:9200/_ilm/policy/cicspa-ilm-policy"
````

**Notes:**

-   This sample policy is designed for use with data streams.
-   The policy contains a rollover action that triggers the creation of a new backing index for the data stream either when the primary shard size reaches 10 GB, or 10 days after the current backing index was created, whichever occurs first.
-   This sample policy deletes indices 20 days after rollover.
-   If you *don't* configure a policy, and you keep forwarding data to this Elasticsearch instance, then you will eventually run out of disk space.
-   The supplied policy is an example only. Configure a policy that matches your requirements and available disk space.
-   For details on creating index lifecycle policies and associating them with indices, see the [Elastic Stack ILM documentation](https://www.elastic.co/guide/en/elasticsearch/reference/current/index-lifecycle-management.html).

### Create an Elasticsearch index template

Use the `elasticsearch/cicspa-index-template.json` file as the body of an Elasticsearch create index template API request.

Example API request:

````sh
curl -X PUT -H "Content-Type: application/json" -d @cicspa-index-template.json "http://localhost:9200/_index_template/cicspa"
````
The index template has only one purpose: to map incoming string fields to the `keyword` data type, rather than the default `text` data type.

### Import Kibana saved objects

Import the saved objects in the `kibana/export.ndjson` file into Kibana.

Either:
-   In the Kibana UI:
    1.  Select Burger Menu Icon ► Stack Management ► Saved Objects ► Import
    2.  Select the following import options:
        -  Check for existing objects
           -  Automatically overwrite conflicts
    3. Click **Import**

-   Use the Kibana import saved objects API

**Tip:** Rather than importing into the default space, import into a Kibana space that you have created specifically for these dashboards.

#### Avoid incomplete lists of terms in Controls dropdowns

With default Kibana settings, depending on the number of documents you have indexed, the list of terms available in a Kibana Controls dropdown might be incomplete. In the same dashboard, you might see terms in charts *that are not available in a Controls dropdown for that field*.

There are [many topics in the Elastic Kibana discussion forum](https://discuss.elastic.co/search?q=terms%20list%20incomplete%20%23elastic-stack%3Akibana) about this issue.

The recommended "fix" (sic, deliberately in quotes): in `$KIBANA_HOME/config/kibana.yml`, set high values for the following settings. For example:

```yaml
kibana.autocompleteTimeout: 5000
kibana.autocompleteTerminateAfter: 10000000
```

**Tip:** Before editing `kibana.yml`, stop Kibana. For example, in a Linux command shell, enter `service kibana stop`. After editing, restart Kibana: enter `service kibana start`.

### Configure Logstash

This repository contains a Logstash pipeline configuration ("config") file, `logstash/pipeline/cicspa-tcp-to-local.conf`, that configures Logstash to listen on a TCP port for JSON Lines data from OMEGAMON Data Provider, and then forwards the data to Elasticsearch on the same computer.

How you use that config file depends on your Elastic Stack installation and your local site practices for configuring Logstash pipelines.

#### Single or multiple Logstash pipelines?

Before proceeding, you need to know whether your instance of Logstash is for use only with this repository or is also used for other purposes, other inputs. Specifically, you need to know whether your use of Logstash involves a *single pipeline* or *multiple pipelines*.

If you have installed a new instance of Elastic Stack as a sandbox environment for testing this repository, then you can use a single Logstash pipeline.

However, if you are using this repository with an existing instance of Elastic Stack that already has other inputs, then it is more likely that you will need to use multiple pipelines.

#### Single pipeline

If your instance of Logstash is for use only with this repository, then you can delete the contents of the default Logstash config directory, and then copy the supplied config file into that directory.

For example:

1.  Delete the contents of the `/etc/logstash/conf.d/` directory.

    The default Logstash pipeline directory path depends on your platform.

2.  Copy the `logstash/pipeline/cicspa-tcp-to-local-elasticsearch.conf` file to the `/etc/logstash/conf.d/` directory.


#### Multiple pipelines

For information about configuring multiple pipelines, see the [Logstash documentation](https://www.elastic.co/guide/en/logstash/7.14/multiple-pipelines.html).

#### Elasticsearch data streams

The supplied Logstash config uses the following `index` option to set Elasticsearch data stream names:

```ruby
index => "cicspa_%{[@metadata][code_identifier]}"
```

where `[@metadata][code_identifier]` is the lowercase of the code field in the incoming JSON data, with a value such as `conectio`.

The sample dashboards use corresponding code-specific data streams, such as `cicspa_conectio*`.

### Refresh the Logstash config

Unless you have configured Logstash to automatically detect new pipeline configurations, stop and then restart Logstash.

For example, to stop the Logstash service on Linux, enter:

```sh
service logstash stop
```

Logstash can take a while to respond to that command (the signal to stop). If the response from that command ends with:

> logstash stop failed; still running.

wait for several seconds, and then enter:

```sh
service logstash status
```

You want to see:

> logstash is not running

Enter:

```sh
service logstash start
```

### Forward the sample data to Logstash

The sample data can be downloaded from the CICS SupportPac "[CA10:CICS Performance Analyzer for z/OS - Output to JSON Lines](https://www.ibm.com/support/pages/node/1282648)".

The dashboards use the `cicspa_sample_data.1.0.0.jsonl`.

Use a TCP forwarder to send the sample data file, `cicspa_sample_data.1.0.0.jsonl`, to the listening TCP port.

For example, via `socat`:

```sh
socat -u cicspa_sample_data.1.0.0.jsonl TCP4:localhost:5046
```

or `ncat`:

```sh
ncat --send-only -v localhost 5046 < omegamon-1-line-test.jsonl
```

### Create a transform for dsa and storage data

The CICS Storage dashboards requires a combination of dsa and storage data. In order to achieve this we need to run a transform.

Use the `elasticsearch/transform-cicspa-dsa-and-storage.json` file as the body of an Elasticsearch create transform API request.

````sh
curl -X PUT -H "Content-Type: application/json" -d @transform-cicspa-dsa-and-storage.json "http://localhost:9200/_transform/transform_cics_dsa_storage"
````

In kibana, go to Stack Management ► Transforms to start the transform_cics_dsa_storage.

### Browse the Kibana dashboards

Use your web browser to go to the following Kibana URL:

```
http://localhost:5601/s/cicspa/app/dashboards#/view/220bf890-3238-11ec-adc8-8d3b57a2f8c2?_g=(time:(from:'2019-08-30T16:00:00.000Z',to:'2019-08-31T15:59:59.999Z'))
```

-   `localhost` assumes that you are using the Elastic Stack installed on your local computer. If that is not the case, replace `localhost` with the hostname or IP address of the computer where you have installed the dashboards.

-   The `time` parameter in the URL specifies the time range of the sample data, so that you do not have to specify this range to Kibana yourself.

-   The dashboard developers recommend the Chrome web browser.

## Deleting the sample data

Suppose that you have installed the sample data. Later, you decide to use the same Elastic Stack instance for your own data. You might want to delete the sample data first.

To delete all Elasticsearch indices for the sample data, send the following Elasticsearch REST API request (for example, using the Dev Tools option in Kibana):

```
DELETE /cicspa*
```