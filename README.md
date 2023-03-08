# SLO percentile - Elasticsearch Kibana

Example of percentile representation to build your SLOs from random latency and Apache logs in Kibana.

![50th, 85th, 95th, and 99th percentile latencies for a system; what you can get following this repo](./percentiles.png)

Pre-requisite :

- Docker
- Docker Compose

## Start Elasticsearch and Kibana

1. Start the mono-node cluster running the following compose command

    ```bash
    docker network create slo_percentile_network
    docker-compose -f cluster.docker-compose.yml up -d
    ```

2. Check Kibana is up and running

    Go to `http://localhost:5601` and login with `elastic`/`changeme`

## Generate and ingest random logs

1. Generate random logs

    Run this script that will generate 30000 lines of HTTP queries.

    ```bash
    docker-compose -f generate.docker-compose.yml up --build -d

    while [ $(docker ps -q -f status=running -f name=^/gen_el_apache$) ]; do echo "Generating..."; sleep 1 ; done
    echo "log|latency|datetime" > ./logs/random_30000.csv
    docker logs gen_el_apache >> ./logs/random_30000.csv
    echo "Finished dumping logs in ./logs/random_30000.csv"
    ```

    :information_source: This script generates latency bursts from time to time

2. Ingest data to Elasticsearch with [elasticsearch-dump](https://github.com/elasticsearch-dump/elasticsearch-dump)

    ```bash
    export INDEX=apache_logs_$(date +%s)
    docker run --rm \
        -v "$(pwd)/logs:/data:ro" \
        --network slo_percentile_network \
        -ti elasticdump/elasticsearch-dump:v6.94.1 \
        --input="/data/mapping.json" \
        --output=http://elastic:changeme@es01:9200/$INDEX \
        --type=mapping
    docker run --rm \
        -v "$(pwd)/logs:/data:ro" \
        --network slo_percentile_network \
        -ti elasticdump/elasticsearch-dump:v6.94.1 \
        --input="csv:///data/random_30000.csv" \
        --output=http://elastic:changeme@es01:9200/$INDEX \
        --limit=1000 \
        --csvDelimiter="|" \
        --type=data
    ```

## Build your percentile dashboard

1. Open Kibana in your browser

    Go to `http://localhost:5601` and login with `elastic`/`changeme`

2. Go to the _Discover_ tab and click "Create data view"

    Fill with :

    - Name : `My logs`
    - Index pattern : `apache_logs_*`
    - Select `datetime` as timestamp field

    Click "Save data view to Kibana"

    Set the time frame to "last 7 days"

    You should be redirect to the view and see logs.

3. Go to the _Visualize Library_ tab and click "Create new visualization" then "Lens"

    Configure it :

    - Name it `50th, 85th, 95th, and 99th percentile latencies for a system`
    - Set visualization type as _Area stacked_
    - Set left axis size option to _Logarithmic_
    - Set the horizontal axis with field `datetime` (function _Date histogram_)
    - Add a vertical axis for each of the percentiles : `50th`, `85th`, `95th`, `99th` (function _Percentile_)

    Click the upper right button "Save and return"

4. You're good to watch !

    ![Editing percentiles visualization](./percentiles_edit.png)
