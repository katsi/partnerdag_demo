# Telraam Traffic Snapshot Live Implementation
1. Set correct permissions for the EPSG database location:
    ```bash
    mkdir -p ./server/tmp/epsg;
    chmod -R 0777 ./server/tmp;
    chmod +x ./server/seed/seed.sh;
    ```

2. Start all systems and wait until they are available (except workbench) using:
    ```bash
    docker compose up -d
    while ! docker logs $(docker ps -q -f "name=ldes-server$") 2> /dev/null | grep 'Cancelled mongock lock daemon' ; do sleep 1; done
    ```

3. Create LDES'es and their views using the [seed script](./server/seed/seed.sh):
    > **Note**: the base address of the server can be specified in the [server config](./server/config.yml) in the `ldes-server.host-name` parameter. In order for the links below to work you need to export the correct base address, e.g.:
    ```bash
    export LDES_SERVER_BASE=http://localhost:8080/ldes
    export LDES_SERVER_ADMIN_BASE=http://localhost:8080/ldes
    ```
    OR (for cloud deploy):
    ```bash
    export LDES_SERVER_BASE=https://telraam-api.net/ldes;
    export LDES_SERVER_ADMIN_BASE=http://ldes-1840995827.eu-central-1.elb.amazonaws.com/ldes
    export LDES_SERVER_CURL_CUSTOM_HEADER='-H "X-Ldes-Security=IuB9phee8eseiphoh3thieY9Ahpeelai"'
    ```

    > **Note**: the LDES server has an API that allows to dynamically create and remove data collections, views and the metadata. The LDES server API documentation is available at: 
    > * admin: http://localhost:8080/ldes/v1/swagger-ui/index.html?urls.primaryName=admin
    > * base:  http://localhost:8080/ldes/v1/swagger-ui/index.html?urls.primaryName=base

    ```bash
    sh ./server/seed/seed.sh
    sh ./server/seed/seed.metadata.sh
    ```
    > This will create the following LDES'es and views:
    ```bash
    curl $LDES_SERVER_BASE/observations
    curl $LDES_SERVER_BASE/observations/by-location
    curl $LDES_SERVER_BASE/observations/by-page
    curl $LDES_SERVER_BASE/observations/by-time
    ```

    > **Note**: to allow for discoverability of the LDES'es and their views, you need to create the [metadata information](./server/seed/dcat/) for the catalog and each LDES or view, [upload it](./server/seed/seed.metadata.sh) to the LDES server and configure the metadata crawler to query the LDES server [root URL](http://localhost:8080).

    > **Note**: the minimum that needs to be provided (as DCAT) is a title and a description for your catalog, LDES'es and views. The metadata can contain a lot more data. You can find more information [here](https://www.vlaanderen.be/geopunt/vlaams-geoportaal/metadata/metadata-in-vlaanderen) (in Dutch).

4. Start the LDIO workbench
    ```bash
    docker compose up ldio-workbench -d
    while ! docker logs $(docker ps -q -f "name=ldio-workbench$") 2> /dev/null | grep 'Started Application in' ; do sleep 1; done
    ```

    > **NOTE**: Telraam will push the new state hourly to the workbench HTTP endpoint, but if needed you can get the current state from https://telraam-api.net/v1/reports/traffic_snapshot_live passing an API key. This is saved to ./server/tmp/data.json.

Push the current state to the workbench pipeline:
```bash
curl -H "Content-Type: application/json" http://localhost:8081/traffic-pipeline -d @./server/tmp/data.json
```

Or push only the LDES data of one province:
```bash
curl -H "Content-Type: application/json" http://localhost:8081/traffic-pipeline -d @./server/data/Antwerpen_features.json
```

5. To verify, follow the LDES starting at the view:
    ```bash
    curl $LDES_SERVER_BASE/observations/by-page?pageNumber=1
    curl $LDES_SERVER_BASE/observations/by-location?tile=0/0/0
    ```

6. To end:
    ```bash
    docker compose rm ldio-workbench --stop --force --volumes
    docker compose down
    ```
