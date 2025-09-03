# The Indicium Tech Way of Meltano

Welcome to the Indicium Tech Way of Meltano! This guide navigates the use of Meltano as recommended by Squad 42 team at Indicium Tech.

Wherever our approach diverges from the suggestions in the [official Meltano documentation](https://docs.meltano.com), we will emphasize the difference and provide our rationale.

Although Meltano offers features for constructing an entire data pipeline, from extraction to data transformation and orchestration, we at Indicium Tech tend to favor other tools that handle some of these tasks more efficiently, such as dbt and Airflow. Hence, we primarily use Meltano for [data integration](https://docs.meltano.com/getting-started/part2#run-your-data-integration-el-pipeline) (extract and load), barring certain specific cases where we state otherwise.

![Python](https://img.shields.io/badge/python-3670A0?style=for-the-badge&logo=python&logoColor=ffdd54)

[TOC]

## 1. Recipes

If you're in a "shut up and run my pipelines" mood, choose the recipe that best suits your needs from the options below and customize it accordingly.

First, initiate a Meltano project with `git clone git@bitbucket.org:indiciumtech/meltano_base_project.git`.

### 1.1. Postgres DB to an S3 bucket

- Set your `plugins/extractors/extractors_config.yml` as follows:

``` yaml
plugins:
  extractors:
  - name: tap-postgres
    config:
      host: <your_postgres_host>
      user: <your_postgres_user>
      password: ${POSTGRES_PASSWORD} # define POSTGRES_PASSWORD in .env file: best practice
      dbname: <your_postgres_database>
      filter_schemas: <your_desired_schemas_comma_separated>
      default_replication_method: FULL_TABLE
```

- Set your `plugins/loaders/loaders_config.yml` as follows:

``` yaml
plugins:
  loaders:
  - name: target-s3-csv
    config:
      aws_access_key_id: ${AWS_ACCESS_KEY_ID} # define in .env file
      s3_bucket: <your_s3_bucket>
      aws_region: <your_aws_region>
      aws_secret_access_key: ${AWS_SECRET_ACCESS_KEY} # define in .env file
      naming_convention: output/{date}/{stream}.csv
```

After that, execute the following:

``` bash
meltano install # installs requested plugins 

meltano run tap-postgres target-s3-csv # voila

```

*Note: if you want to integrate only a specific table, say `orders` table from the `public` schema, add a `select` property to your extractor configuration:*

``` yaml
plugins:
  extractors:
  - name: tap-postgres
    config:
      host: <your_postgres_host>
      user: <your_postgres_user>
      password: ${POSTGRES_PASSWORD} # define POSTGRES_PASSWORD in .env file: best practice
      dbname: <your_postgres_database>
      filter_schemas: <your_desired_schemas_comma_separated>
      default_replication_method: FULL_TABLE
    select:
      - public-orders.*
```

### 1.2. S3 bucket to a Postgres DB

- Set your `plugins/extractors/extractors_config.yml` as follows:

``` yaml
plugins:
  extractors:
    - name: tap-s3-csv
        config:
          aws_access_key_id: ${AWS_ACCESS_KEY_ID}
          aws_secret_access_key: ${AWS_SECRET_ACCESS_KEY}
          bucket: <your_s3_bucket>
          start_date: <desired_start_date>
          tables:
            - table_name: <a_table_name> # e.g. orders
              search_pattern: <table_data_files_matching_pattern> # e.g. data/*orders*.csv
              key_properties: <list_table_primary_keys> # e.g. ["id"]
            - table_name: <another_table_name>
              # ...
            # ...
```

- Set your `plugins/loaders/loaders_config.yml` as follows:

``` yaml
plugins:
  loaders:
    - name: target-postgres
      variant: meltanolabs # much easier to setup than default variant
      config:
        add_record_metadata: <true_or_false>
        flattening_enabled: <true_or_false>
        host: <output_db_host>
        user: <output_db_user>
        password: ${POSTGRES_PASSWORD}
        database: <output_database>
        default_target_schema: <desired_output_schema>

```

After that, execute the following:

``` bash
meltano install # installs requested plugins 

meltano run tap-s3-csv target-postgres # voila

```

## 2. Project Structure

To initialize a new Meltano project the Indicium Tech way, clone our base project:

``` bash
git clone git@bitbucket.org:indiciumtech/meltano_base_project.git
```

*Note: The [official Meltano documentation](https://docs.meltano.com/concepts/project) advises using the `meltano init` command to initiate a project. However, this leads to a complex project structure that doesn't align with our purpose for using Meltano at Indicium.*

This process generates a new base project with the following file tree:

``` bash
    ./
    ├── bitbucket-pipelines.yml
    ├── create_custom.sh
    ├── Dockerfile
    ├── DOCS.md
    ├── env_config.yml
    ├── meltano.yml
    ├── pipelines/
    │   └── example_pipeline/
    │       └── example_config.yml
    ├── plugins/
    │   ├── custom/
    │   │   └── custom_config.yml
    │   ├── extractors/
    │   │   └── extractors_config.yml
    │   ├── loaders/
    │   │   └── loaders_config.yml
    │   └── mappers/
    │       └── mappers_config.yml
    ├── README.md
    ├── requirements.txt
```

### 2.1. `meltano.yml`

Every Meltano project requires a `meltano.yml` file. This file holds your project configuration and signifies to Meltano that a specific directory is a Meltano project. The`version` is the only obligatory property, which should always be `1`.

*Note: The official Meltano documentation recommends using the Meltano CLI to configure the entire Meltano project and run pipelines, resulting in a large `meltano.yml` configuration file, making it challenging to locate a specific configuration for modification. We, however, utilize the `include_paths` directive to foster a more modular and intuitive setup for managing distinct parts of the project configuration.*

A crucial configuration for Meltano projects is the  `state_backend`, which determines where your pipeline states for incremental replication will be stored and retrieved by Meltano.

By default, the files will be stored and handled inside the `.meltano` directory, and you can leave it as it is for development purposes. However, you might want to store these states in a cloud environment for your production pipelines. To achieve this, you should uncomment the corresponding `state_backend` section on the `meltano.yml` file.

For instance, if you are using an S3 bucket, your `meltano.yml` should look like this:

``` yaml
# meltano.yml
version: 1
include_paths:
  - env_config.yml
  - ./pipelines/**/*.yml
  - ./plugins/custom/custom_config.yml
  - ./plugins/extractors/extractors_config.yml
  - ./plugins/loaders/loaders_config.yml
  - ./plugins/mappers/mappers_config.yml
state_backend:
  uri: s3://<bucket-name>/<prefix for state JSON blobs>
  s3:
    aws_access_key_id: ${AWS_KEY_ID_STATE_BACKEND}
    aws_secret_access_key: ${AWS_SECRET_STATE_BACKEND}
```

*Note: DO NOT include empty files or files with only commented code! If you try to include an 'empty' file, Meltano will raise a generic error.*

### 2.2. `requirements.txt`

This file will contain the appropriate extra necessary to set up the cloud state backend. For instance, taking into account the above example of an S3 state backend, this fill would read:

``` text
# requirements.txt
meltano[s3]
```

### 2.3. `env_config.yml`

In this file, we define Meltano environments to run our pipelines. The main use of these environments is to define environment variables that will be accessed by the plugins in a pipeline at runtime.

The following example illustrates a case where we can change the output database and state backend by simply passing a different environment when running pipelines.

The `default_environment` property defines which environment will be passed to pipelines in case the user does not specify one.

``` yaml
default_environment: dev      
environments:
  # Creation of environments
  - name: dev
    # 'env' indentation receives variables to export to meltano pipelines
    env:
      #To Use environment variables coming from .env or another source use
      OUT_DB_USER: ${DEV_DB_USER}
      OUT_DB_PASSWORD: ${DEV_DB_PASSWORD}
      OUT_DB_NAME: ${DEV_DB_NAME}
      #For Azure State Backend
      AZURE_CONN_STATE_BACKEND: ${AZURE_CONNECTION_STRING_STATE_BACKEND_DEV}

  - name: prod
    env:
      #Place your 'prod' variables here
      OUT_DB_USER: ${PROD_DB_USER}
      DB_PASSWORD: ${PROD_DB_PASSWORD}
      OUT_DB_NAME: ${PROD_DB_NAME}
      #For Azure State Backend
      AZURE_CONN_STATE_BACKEND: ${AZURE_CONNECTION_STRING_STATE_BACKEND_PROD}
```

### 2.4. `plugins/`

This directory contains the base building blocks of our data integration pipelines: plugins.

We separate each category into its folder: `extractors` (or taps), `loaders` (or targets), and `mappers` (inline data transformers). The `custom` folder can contain each of these plugin types but for plugins created by the user. The most common case for us will be [custom extractors](#3-custom-extractors).

#### 2.4.1. `extractors/`

Here we define the base plugins that will extract data from sources and set them in a stream for being redirected to a target.

Here is an example of a base extractor plugin definition:

``` yaml
# ./plugins/extractors/extractors_config.yml
plugins:
  extractors:
  - name: tap-postgres
    variant: meltanolabs
    pip_url: pipelinewise-tap-postgres
    config:
      host: ${VAR_IN_DB_HOST}
      user: ${VAR_IN_DB_USER}
      password: ${VAR_IN_DB_PASSWORD}
      dbname: ${VAR_IN_DB_NAME}
      default_replication_method: FULL_TABLE
```

#### 2.4.2. `loaders/`

This kind of plugin will direct the data in the streams to a target destination.

Here is an example of a base loader plugin definition:

``` yaml
# ./plugins/loaders/loaders_config.yml
plugins:
  loaders:
  - name: target-postgres
    variant: transferwise
    pip_url: pipelinewise-target-postgres
    config:
      host: ${VAR_OUT_DB_HOST}
      user: ${VAR_OUT_DB_USER}
      password: ${VAR_OUT_DB_PASSWORD}
      dbname: ${VAR_OUT_DB_NAME}
      default_target_schema: public
      hard_delete: true

  - name: target-jsonl
    variant: andyh1203
    pip_url: target-jsonl
  - name: target-s3
    variant: crowemi
    pip_url: git+https://github.com/crowemi/target-s3.git
```

#### 2.4.3. `mappers/`

You can manipulate or transform data after extraction and before loading through mappers. These mappers find common application in:

- Aliasing streams or properties to customize naming downstream.
- Filtering stream records based on any logic you define.
- Transforming properties inline, for instance, converting types or sanitizing PII data.
- Removing properties from the stream.
- Adding new properties to the stream.

Consider the following example that demonstrates the setup of base mapper plugins:

``` yaml
# ./plugins/mappers/mappers_config.yml
mappers:
- name: meltano-map-transformer
    pip_url: git+https://github.com/MeltanoLabs/meltano-map-transform.git
    executable: meltano-map-transform
```

### 2.5. `pipelines/`

This folder is where our pipelines `per se` are defined. We recommend using one sub-folder for each pipeline, and in each folder define a `job`, which essentially is an alias (or `name`) for a sequence of `tasks`.

In addition, we customize each plugin for use in a pipeline with an `inherit_from` directive. As the name suggests, plugins defined this way inherit all configurations from the base objects in the `plugins` folder and have additional configurations for the specific pipeline we are defining.

Here is an example that illustrates all these concepts:

``` yaml
plugins:
  extractors:
  - name: tap-postgres--orders
    inherit_from: tap-postgres
    config:
      port: ${VAR_IN_DB_PORT}
    select:
    - public-orders.*
  
  mappers:
  - name: add-extracted-at
    inherit_from: transform-field
    config:
      stream_maps:
        public-orders:
          extracted_at: "datetime.datetime.now(datetime.timezone.utc)"

  loaders:
  - name: target-postgres--orders
    inherit_from: target-postgres
    config:
      port: ${VAR_OUT_DB_PORT}

jobs:
- name: orders-pipeline
  tasks:
  - tap-postgres--orders add-extracted-at target-postgres--orders
```

In this example, the base tap-postgres in the `plugins` folder extracts data from all tables of the `public` schema to put in a stream. However, after the `inherit_from` clause and the configuration `select`, only the `orders` table is streamlined in the `orders-pipeline` job (smart, hum?).

Once we have defined a pipeline (such as the `orders-pipeline` from the previous example), we can execute it via:

``` bash
# using the default environment
meltano run orders-pipeline

# or

# using a predefined environment in env_config.yml, e.g. 'prod'

# meltano --environment=prod run  orders-pipeline
```

These are the 20% of the Indicium Way of Meltano that you will need to execute 80% of the work you probably want to do with Meltano at Indicium.

For the sake of completeness, let us comment on the purpose of the remaining files present in the base project structure:

- `bitbucket-pipelines.yml`: skeleton CI/CD pipeline that you might be interested to build upon for your production setup;

- `Dockerfile`: basic setup for building a Meltano image with all plugins and pipelines from your project to be used in production, e.g. in a `DockerOperator` in Airflow.

- `.gitignore`, `.dockerignore`: pre-defined files you might want to ignore for use in the respective tools.

- `create_custom.sh`: simple bash script that helps in the workflow of developing [custom-extractors](#3-custom-extractors).

## 3. Custom Extractors

Custom extractors are tools or scripts developed to retrieve data from non-standard data sources like custom databases or SaaS APIs, such as Appwrite. They convert this data into a suitable format for loading into a target destination, such as a data warehouse. These extractors, known as taps in the Singer framework, do not come as integral parts of the tool in use.

Singer taps and targets can streamline the use of custom extractors. Singer, a popular data extraction tool, offers specifications for creating extractors and loaders. In Singer's context, a custom extractor is a tap tailored to meet an organization's specific requirements.

Operating taps and targets manually can be labor-intensive. Meltano extractor/loader plugins offer a more efficient approach. Meltano's EL (Extract and Load) capabilities manage the complexities of configuration, stream discovery, and state management associated with Singer.

### 3.1. Creating Custom Extractors

The following steps outline how to create a custom extractor for a Meltano project, according to the Indicium Tech way:

#### 3.1.1. Create a Project Using the Cookiecutter Template

Run the following commands at the root of your Meltano project:

``` bash
chmod +x create_custom.sh
bash create_custom.sh
```

These commands will prompt you to set up your project.

As a result, a new directory, tap-<source_name>, will appear in plugins/custom, containing the foundational code for your tap development, along with a `meltano.yml` file that you can use to test your custom extractor.

*Note: The `meltano.yml` file within the tap folder informs Meltano that the folder is also a Meltano project. This setup lets you test your plugin in isolation, without impacting your original Meltano project. Once your plugin behaves as expected, you can shift the directory back to the project root and use your new plugin as discussed earlier.*

Inside the tap folder, the main files we will be using in the development of the custom extractor are:

- `tap.py`: defines the basic tap configurations and the streams it will create when interacting with the data source;

- `client.py`: defines the basic stream class and objects necessary for interaction with the source system, like authentication, pagination, and base stream objects.

- `streams.py`: implements the stream objects to be returned by the tap by adapting the base objects in `client.py` for each API endpoint.

#### 3.1.2. Configure the Custom Extractor to Consume Data from the Source

This step involves adapting the extractor to access data from the preferred source.

While the specific implementation details can vary significantly depending on the API, the following steps are generally common across all implementations:

- Define tap configurations: In the `tap.py` file, you define your `Tap<source_name>` class. The `config_jsonschema` property of this object outlines the configuration parameters you will provide for your tap. Remember, this definition of tap configurations happens "in a vacuum", meaning you should provide these configurations for the tap to run in isolation.

*Note: When integrating with Meltano, you should define these same configurations in the `settings` property of the base plugin description. Keep in mind that Meltano won't be aware of the configurations you define at the `tap.py` level unless you specify them in the `settings` property of the plugin.*

- Identify the streams you need to replicate: In the Tap class, you need to implement the `discover_streams` method by returning a list of streams you want to extract data from.

The following example illustrates how to implement this:

``` python
# ./plugins/custom/tap_okta/tap.py
"""okta tap class."""

from __future__ import annotations

from singer_sdk import Tap
from singer_sdk import typing as th  # JSON schema typing helpers

# TODO: Import your custom stream types here:
from tap_okta import streams


class Tapokta(Tap):
    """okta tap class."""

    name = "tap-okta"

    # TODO: Update this section with the actual config values you expect:
    config_jsonschema = th.PropertiesList(
        th.Property(
            "auth_token",
            th.StringType,
            required=True,
            secret=True,  # Flag config as protected.
            description="The token to authenticate against the API service",
        ),
        th.Property(
            "okta_domain",
            th.StringType,
            required=True,
            description="Organization's unique subdomain in okta.",
        ),
        th.Property(
            "performance_optimization",
            th.BooleanType,
            description="Complex DelAuth configurations may degrade performance when fetching specific parts of the response, and passing this parameter can omit these parts, bypassing the bottleneck.",
        ),
        th.Property(
            "search",
            th.StringType,
            description="Searches for users with a supported filtering expression for most properties. Okta recommends this option for optimal performance.",
            examples=[
                'status eq "STAGED"',
                "lastUpdated gt \"yyyy-MM-dd'T'HH:mm:ss.SSSZ\"",
                'id eq "00u1ero7vZFVEIYLWPBN"',
                'type.id eq "otyfnjfba4ye7pgjB0g4"',
                'profile.department eq "Engineering"',
                'profile.occupation eq "Leader"',
                'profile.lastName sw "Smi"',
            ],
        ),
        th.Property(
            "limit",
            th.IntegerType,
            description="Specifies the number of results returned (maximum 200). If you don't specify a value for limit, the maximum (200) is used as a default.",
        ),
        th.Property(
            "sortBy",
            th.StringType,
            description="Specifies field to sort by (for search queries only).",
        ),
        th.Property(
            "sortOrder",
            th.StringType(allowed_values=["asc", "desc"]),
            description="Specifies sort order asc or desc (for search queries only). Sorting is done in ASCII sort order (that is, by ASCII character value), but isn't case sensitive. sortOrder is ignored if sortBy is not present, is optional and defaults to ascending.",
        ),
    ).to_dict()

    def discover_streams(self) -> list[streams.oktaStream]:
        """Return a list of discovered streams.

        Returns:
            A list of discovered streams.
        """
        return [
            streams.UsersStream(self),
        ]


if __name__ == "__main__":
    Tapokta.cli()
```

- Specify the base API URL: This is the `url_base` property of the `<source_name>Stream` class in the `client.py` file.

``` python
# ./plugins/custom/tap_okta/client.py

# ... 

class oktaStream(RESTStream):
    """okta stream class."""

    @property
    def url_base(self) -> str:
        """Return the API URL root, configurable via tap settings."""
        # TODO: hardcode a value here, or retrieve it from self.config
        return f"https://{self.config.get('okta_domain')}"

# ...

```

- Outline the streams' schemas: You should define the streams you intend to use in the `streams.py` file. Pay particular attention to the `path` property. This gets appended to the base URL to determine the streams' endpoint in the API. Define the stream's schema as well. While you can do this directly within the class (similar to the `config_jsonschema` property of the tap's class), we recommend setting the `schema_filepath` and creating a `schemas` folder within the tap directory to store the schema files.

``` python
# ./plugins/custom/tap_okta/streams.py

# ...

SCHEMAS_DIR = Path(__file__).parent / Path("./schemas")
  
class UsersStream(oktaStream):
    """Define custom stream."""

    name = "users"
    path = "/api/v1/users"
   
    schema_filepath = SCHEMAS_DIR / "events.json" 

# ...

```

Please note, by default, RESTStreams presume the REST method is `GET`. Typically, you set up the `get_url_params` method to adapt the endpoint for querying purposes. However, if the API uses a `POST` method, you should set your stream's `rest_method` attribute to `POST` and overwrite the `prepare_request_payload` method.

``` python
# ./plugins/custom/tap_totango/streams.py

# ...

class UsersStream(totangoStream):
    """Define custom stream."""

    name = "users"
    rest_method = "POST"

    path = "/api/v1/search/users"

    schema_filepath = SCHEMAS_DIR / "users.json"

    def prepare_request_payload(
        self,
        context: dict | None,  # noqa: ARG002
        next_page_token: t.Any | None,  # noqa: ARG002
    ) -> dict | None:
        """Prepare the data payload for the REST API request.

        By default, no payload will be sent (return None).

        Args:
            context: The stream context.
            next_page_token: The next page index or value.

        Returns:
            A dictionary with the JSON body for a POST requests.
        """
        # TODO: Delete this method if no payload is required. (Most REST APIs.)
        params = self.config
        query = {
            "terms": params["users_terms"],
            "fields": params["users_fields"],
            "offset": params["users_offset"],
            "count": params["users_count"],
            "sort_by": params["users_sort_by"],
            "sort_order": params["users_sort_order"],
        }
        data = {"query": json.dumps(query)}
        return data

# ...

```

#### 3.1.3. Test The Newly Created Tap

Navigate to the tap folder and configure your plugins' `settings` to match the ones you defined. Next, run `meltano install`.

 After that, run `meltano run tap-<source_name> target-jsonl` and check the result in the `output` folder that will be created.

### 3.2. Incremental Replication Implementation

Incremental replication is a vital feature in data integration pipelines. It optimizes the extraction time and resource usage by only extracting modified data. The Meltano SDK offers incremental replication mechanisms for three types of implementation, based on the API's filtering capabilities.

Remember, for users familiar with the Singer specification, we're aware of the "at least once" method for incremental extractions. Nonetheless, we deliberately implement incremental extractions using a "greater than" comparison, believing the benefits of eliminating duplicate data outweigh the potential downsides.

*Note [for users familiar with the Singer spec]: we are aware of the [at least once](https://sdk.meltano.com/en/latest/implementation/at_least_once.html) method for incremental extractions. Nonetheless, we deliberately implement incremental extractions using a "greater than" comparison, believing the benefits of eliminating duplicate data outweigh the potential downsides.*

#### 3.2.1. Filter Mechanism in Request's URL

If the API offer filtering capabilities as query parameters in the URL (which most APIs of practical interest do), you can take advantage and request the incremental records directly at the source by overwriting the `get_url_params` method, as the following example demonstrates:

``` python
# ./plugins/custom/tap_okta/streams.py

# ...

class UsersStream(oktaStream):

# ...

    def get_url_params(
        self, context: dict | None, next_page_url: str | None
    ) -> dict[str, Any]:
        
        # ...

        if self.replication_method == "INCREMENTAL":
            bookmark = self.get_starting_timestamp(context)
            if bookmark:
                # convert date to expected format for querying the API
                bookmark_date = str(bookmark).split("+")[0] + ".000Z"

                url_params["search"] = f'{self.replication_key} gt "{bookmark_date}"'
        
        # ...

# ...
```

#### 3.2.2. Filter Mechanism in Request's Body

This is analogous to the previous implementation but for streams that have a `POST`  rest method. Once again, we can take advantage and request the incremental records directly at the source, but this time overwriting the `prepare_request_payload` method, as in the following example:

``` python
# ./plugins/custom/tap_totango/streams.py

# ...

class EventsStream(totangoStream):

# ...

    def prepare_request_payload(
        self,
        context: dict | None,  # noqa: ARG002
        next_page_token: t.Any | None,  # noqa: ARG002
    ) -> dict | None:
        # ...

        if self.replication_method == "INCREMENTAL":
            bookmark = self.get_starting_replication_key_value(context)
            if bookmark:
                # in this case, the date value is a timestamp in Unix time (EPOCH) milliseconds format.
                payload["terms"] = [
                    {
                        "type": "date", 
                        "term": {self.replication_key}, 
                        "gte": {bookmark},
                                    
                    },
                ]

        
        # ...
        
        # ...

# ...
```

#### 3.2.3. No API Filtering Mechanism

If the API doesn't provide a filtering mechanism, request the complete object collection and parse the response to only stream the incremental records. You can accomplish this within the `parse_response` method. Overwriting the `request_records` method is necessary to provide the `context` dictionary to the `parse_response` method, thereby giving it access to the tap's state.

``` python
# ./plugins/custom/tap_datbricksops/streams.py

# ...

from tap_databricksops.client import DatabricksOpsStream
from singer_sdk.helpers.jsonpath import extract_jsonpath
from singer_sdk import metrics

# ...

class Jobs(DatabricksOpsStream):
    
    # ...

    def parse_response(
            self, response: Response, context: dict | None
        ) -> Iterable[dict]:
            """Parse the response and return an iterator of result records.

            Args:
                response: A raw `requests.Response`_ object.

            Yields:
                One item for every item found in the response.

            .. _requests.Response:
                https://requests.readthedocs.io/en/latest/api/#requests.Response
            """
            records = list(extract_jsonpath(self.records_jsonpath, input=response.json()))
            # bookmark should be a milliseconds epoch
            bookmark = self.get_starting_replication_key_value(context)
            if self.replication_method == "INCREMENTAL" and bookmark:
                incremental_records = []
                for record in records:
                    if record[self.replication_key] > bookmark:
                        incremental_records.append(record)
                yield from incremental_records
            else:
                yield from records

    def request_records(self, context: dict | None) -> Iterable[dict]:
        """Request records from REST endpoint(s), returning response records.

        If pagination is detected, pages will be recursed automatically.

        Args:
            context: Stream partition or context dictionary.

        Yields:
            An item for every record in the response.
        """
        paginator = self.get_new_paginator()
        decorated_request = self.request_decorator(self._request)

        with metrics.http_request_counter(self.name, self.path) as request_counter:
            request_counter.context = context

            while not paginator.finished:
                prepared_request = self.prepare_request(
                    context,
                    next_page_token=paginator.current_value,
                )
                resp = decorated_request(prepared_request, context)
                request_counter.increment()
                self.update_sync_costs(prepared_request, resp, context)
                yield from self.parse_response(resp, context)

                paginator.advance(resp)
```

### 3.3. Parent-Child Streams

The Tap SDK supports parent-child streams, by which one stream type can be declared to be a parent to another stream, and the child stream will automatically receive `context` from a parent record each time the child stream is invoked.

We recommend the following approach to set up this stream configuration:

- Set `parent_stream_type` in the child-stream’s class to the class of the parent.

- Override the `get_child_context` method to return a new child context object based on records and any existing context from the parent stream.

- Use values from the parent context using the keys defined in the previous step.

Here is an abbreviated example that uses the above techniques. In this example, EventsStream is a child of AccountsStream.

``` python
# ./plugins/custom/tap_totango/streams.py

# ...

class AccountsStream(totangoStream):
    """Define custom stream."""

    name = "accounts"
    
    # ...

    def get_child_context(self, record: dict, context: t.Optional[dict]) -> dict:
        """Return a context dictionary for child streams."""
        return {
            "account_id": record["name"],
        }
    
    # ...

# ...

class EventsStream(totangoStream):
    """Define custom stream."""

    name = "events"
    
    # ...

    def prepare_request_payload(
        self,
        context: dict | None,  # noqa: ARG002
        next_page_token: t.Any | None,  # noqa: ARG002
    ) -> dict | None:
        
        # ...

        payload["account_id"] = context["account_id"]
        
        # ...

# ...
```

### 3.4. Capabilities

When defining your custom plugin to operate with Meltano, it's essential to incorporate the following capabilities for the reasons specified:

- `state`: This is a mandatory capability that facilitates incremental replication.

- `catalog`: This capability is necessary for enabling the selection of fields and streams.

- `discover`: This feature allows Meltano to generate the tap's catalog using the --discover flag.

- `stream-maps`: This capability permits the plugin to execute inline transformations prior to transmitting data to a target.

## 4. Continous Integration Tests

Continuous Integration (CI) tests are a critical component of the Continuous Integration and Continuous Delivery (CI/CD) process in software development. CI is a practice that involves automatically integrating code changes from multiple contributors into a shared repository multiple times a day. The purpose of CI is to identify and address integration issues early in the development process, promoting code stability and reducing the likelihood of defects in the final product.

CI tests are automated tests that are executed as part of the CI process. These tests help ensure that the code changes introduced by developers do not break the existing functionality of the software and are compatible with other parts of the codebase.

In Meltano projects, we utilize a Meltano pipeline to do the CI tests for all the pipelines we use. To do this, we create a pipeline with the execution of all the pipelines we want to include in the tests, as in the example below:

```yml
# ./pipelines/ci_pipeline.yml

# ...

jobs:
- name: ci-pipeline
  tasks:
  - tap-totango--accounts target-snowflake--totango # totango-pipeline--accounts
  - tap-totango--events target-snowflake--totango # totango-pipeline--events
  - tap-totango--users target-snowflake--totango # totango-pipeline--users
  - tap-totango--touchpoint_types target-snowflake--totango # totango-pipeline--touchpoint_types
  - tap-totango--touchpoint_tags target-snowflake--totango # totango-pipeline--touchpoint_tags
  - tap-totango--touchpoints target-snowflake--totango # totango-pipeline--touchpoints
  - tap-google-analytics--web_traffic target-snowflake--ga4 # google-analytics-pipeline--web_traffic
  - tap-google-analytics--page_path target-snowflake--ga4 # google-analytics-pipeline--page_path
  - tap-cvent--events target-snowflake--cvent # cvent-pipeline
  - tap-okta--users target-snowflake--okta # okta-pipeline
  - tap-spreadsheets-anywhere--jenkins target-snowflake--jenkins # jenkins-pipeline
  - tap-spreadsheets-anywhere--checkmarx_projects target-snowflake--checkmarx # projects-pipeline
  - tap-spreadsheets-anywhere--checkmarx_findings target-snowflake--checkmarx # findings-pipeline
  - tap-okta--users target-snowflake--okta # sf-pipeline
  - tap-google-search-console--page target-snowflake--gsc # gsc-pipeline--page
  - tap-google-search-console--date target-snowflake--gsc # gsc-pipeline--date
  - tap-google-search-console--device target-snowflake--gsc # gsc-pipeline--device
  - tap-google-search-console--query target-snowflake--gsc # gsc-pipeline--query
```

In this case, we are considering the taps of Totango, Okta, CVENT, GA4 and others. 

It's preferable to have a developer environment to do these tests, both in the Meltano project and in the target tool, which in the case of the example is Snowflake. This way, the data will be sent to clones of the production tables, without affecting the original ones. 

The distinction between environments is made in the env_config.yml file, where we define values for the environment variables **prod** and **dev**.

```yml
# ./env_config.yml

# ...

environments:
  # Creation of environments
- name: dev
  env:
    TARGET_S3_CLOUD_PROVIDER_AWS_AWS_ACCESS_KEY_ID: ${AWS_ID}
    TARGET_S3_CLOUD_PROVIDER_AWS_AWS_SECRET_ACCESS_KEY: ${AWS_PSW}
    AWS_ACCESS_KEY_ID: ${AWS_ID}
    AWS_SECRET_ACCESS_KEY: ${AWS_PSW}
    AWS_KEY_ID_STATE_BACKEND: ${AWS_ID}
    AWS_SECRET_STATE_BACKEND: ${AWS_PSW}
    START_DATE_TAPGA4: ${START_DATE_TAPGA4_DEV}
    START_DATE_TAPOKTA: ${START_DATE_TAPOKTA_DEV}
    START_DATE_TAPGSC: ${START_DATE_TAPGSC_DEV}

    # Snowflake
    SNOWFLAKE_DBNAME: CPP_DATA_SANDBOX
    CVENT_STAGE: CPP_DATA_SANDBOX.CVENT.CVENT_STAGE_DEV
    CVENT_FILE_FORMAT: CPP_DATA_SANDBOX.CVENT.CVENT_FILE_FORMAT
    TOTANGO_STAGE: CPP_DATA_SANDBOX.TOTANGO_API.TOTANGO_STAGE_DEV
    TOTANGO_FILE_FORMAT: CPP_DATA_SANDBOX.TOTANGO_API.TOTANGO_FILE_FORMAT
    OKTA_FILE_FORMAT: CPP_DATA_SANDBOX.OKTA.OKTA_FILE_FORMAT
    OKTA_STAGE: CPP_DATA_SANDBOX.OKTA.OKTA_STAGE_DEV
    GA4_FILE_FORMAT: CPP_DATA_SANDBOX.GA4.GA4_FILE_FORMAT
    GA4_STAGE: CPP_DATA_SANDBOX.GA4.GA4_STAGE_DEV
    JENKINS_FILE_FORMAT: CPP_DATA_SANDBOX.JENKINS_JOBS.JENKINS_JOBS_FILE_FORMAT
    CHECKMARX_FILE_FORMAT: CPP_DATA_SANDBOX.JENKINS_JOBS.CHECKMARX_FILE_FORMAT
    SF_FILE_FORMAT: CPP_DATA_SANDBOX.SALESFORCE_MELTANO.SF_FILE_FORMAT
    SF_STAGE: CPP_DATA_SANDBOX.SALESFORCE_MELTANO.SF_STAGE
    GSC_FILE_FORMAT: CPP_DATA_SANDBOX.GSC_MELTANO.GSC_FILE_FORMAT
    GSC_STAGE: CPP_DATA_SANDBOX.GSC_MELTANO.GSC_STAGE

- name: prod
  env:
    TARGET_S3_CLOUD_PROVIDER_AWS_AWS_ACCESS_KEY_ID: ${AWS_ID}
    TARGET_S3_CLOUD_PROVIDER_AWS_AWS_SECRET_ACCESS_KEY: ${AWS_PSW}
    AWS_ACCESS_KEY_ID: ${AWS_ID}
    AWS_SECRET_ACCESS_KEY: ${AWS_PSW}
    AWS_KEY_ID_STATE_BACKEND: ${AWS_ID}
    AWS_SECRET_STATE_BACKEND: ${AWS_PSW}
    START_DATE_TAPGA4: '2022-01-01T00:00:00Z'
    START_DATE_TAPOKTA: '2022-01-01T00:00:00Z'
    START_DATE_TAPGSC: '2022-01-01T00:00:00Z'

    # Snowflake
    SNOWFLAKE_DBNAME: CPP_DATA
    CVENT_STAGE: CPP_DATA.CVENT.CVENT_STAGE
    CVENT_FILE_FORMAT: CPP_DATA.CVENT.CVENT_FILE_FORMAT
    TOTANGO_FILE_FORMAT: CPP_DATA.TOTANGO_API.TOTANGO_FILE_FORMAT
    TOTANGO_STAGE: CPP_DATA.TOTANGO_API.TOTANGO_STAGE
    OKTA_FILE_FORMAT: CPP_DATA.OKTA.OKTA_FILE_FORMAT
    OKTA_STAGE: CPP_DATA.OKTA.OKTA_STAGE
    GA4_FILE_FORMAT: CPP_DATA.GA4.GA4_FILE_FORMAT
    GA4_STAGE: CPP_DATA.GA4.GA4_STAGE
    JENKINS_FILE_FORMAT: CPP_DATA.JENKINS_JOBS.JENKINS_JOBS_FILE_FORMAT
    CHECKMARX_FILE_FORMAT: CPP_DATA.JENKINS_JOBS.CHECKMARX_FILE_FORMAT
    SF_FILE_FORMAT: CPP_DATA.SALESFORCE_MELTANO.SF_FILE_FORMAT
    SF_STAGE: CPP_DATA.SALESFORCE_MELTANO.SF_STAGE
    GSC_FILE_FORMAT: CPP_DATA.GSC_MELTANO.GSC_FILE_FORMAT
    GSC_STAGE: CPP_DATA.GSC_MELTANO.GSC_STAGE
```

Also note that there are different values for the start date of some taps. In the example, we are setting the start date via an environment variable in the dev environment and a direct assignment in the prod environment. 

In the case of the dev environment, this variable can be set in the CI pipeline itself, either via Github or Bitbucket, and it is suggested to use the day before the current day, i.e. yesterday. This is interesting because we don't want our CI pipeline to last too long, do we? Working with a small amount of data is enough to validate the code and prove its efficiency.

```bash
export START_DATE_TAPGA4_DEV=$(date +%Y-%m-%d -d '1 day ago')
export START_DATE_TAPOKTA_DEV=$(date +%Y-%m-%d -d '1 day ago')
export START_DATE_TAPGSC_DEV=$(date +%Y-%m-%d -d '1 day ago')
```

Finally, for this CI pipeline made in Meltano to work, we need to implement a CI pipeline from the platform where the repository is hosted. 

**Example of the CI pipeline on GitHub**

```yml
# ./.github/workflows/ci_pipeline.yml

# ...

name: Meltano CI

on:
  pull_request:
    branches:
      - main

jobs:
  test-pipelines:
    name: Testing the Pipelines
    runs-on: ubuntu-latest
    env:
      TOTANGO_AUTH_TOKEN: ${{ secrets.TOTANGO_AUTH_TOKEN }}
      OKTA_AUTH_TOKEN: ${{ secrets.OKTA_AUTH_TOKEN }}
      TARGET_SNOWFLAKE_USER: ${{ secrets.TARGET_SNOWFLAKE_USER }}
      TARGET_SNOWFLAKE_PASSWORD: ${{ secrets.TARGET_SNOWFLAKE_PASSWORD }}
      SNOWFLAKE_ROLE: ${{ secrets.SNOWFLAKE_ROLE }}
      SNOWFLAKE_WAREHOUSE: ${{ secrets.SNOWFLAKE_WAREHOUSE }}
      AWS_ID: ${{ secrets.AWS_ID }}
      AWS_PSW: ${{ secrets.AWS_PSW }}
      CVENT_OAUTH_KEY: ${{ secrets.CVENT_OAUTH_KEY }}
      TAP_GA4_PROPERTY_ID: ${{ secrets.TAP_GA4_PROPERTY_ID }}
      TAP_GA4_CLIENT_SECRETS: ${{ secrets.TAP_GA4_CLIENT_SECRETS }}
      GSC_PRIVATE_KEY: ${{ secrets.GSC_PRIVATE_KEY }}
      GSC_CLIENT_EMAIL: ${{ secrets.GSC_CLIENT_EMAIL }}
      
    steps:
      - name: Checkout the repo
        uses: actions/checkout@v3

      - name: Set up Python
        uses: actions/setup-python@v2
        with:
          python-version: 3.8

      - name: Cache Dependencies
        uses: actions/cache@v2
        with:
          path: ~/.cache/pip
          key: ${{ runner.os }}-pip-${{ hashFiles('requirements.txt') }}
          restore-keys: |
            ${{ runner.os }}-pip-

      - name: Installing Meltano and Dependencies
        run: |
          pip install meltano==2.19.1
          pip install -r requirements.txt
          meltano install

      # Testing if every pipeline is working as expected
      - name: Testing All the Pipelines
        run: |
          export START_DATE_TAPGA4_DEV=$(date +%Y-%m-%d -d '1 day ago')
          export START_DATE_TAPOKTA_DEV=$(date +%Y-%m-%d -d '1 day ago')
          export START_DATE_TAPGSC_DEV=$(date +%Y-%m-%d -d '1 day ago')
          meltano --environment=dev run ci-pipeline
          
  build-docker-image:
    name: Building Docker Image
    runs-on: ubuntu-latest
    needs: test-pipelines
    
    steps:
      - name: Checkout Code
        uses: actions/checkout@v2
  
      # Building docker image
      - name: Build Docker Image
        run: |
          docker build -t meltano_test .
        
```

This pipeline will run every time there is a Pull Request to the main branch, thus checking for errors in the code. Note that the meltano run command was done with the environment flag set to dev. 

**Example of the CI pipeline on BitBucket**

```yml
# ./bitbucket-pipelines.yml

# ...

options:
  docker: true

image: meltano/meltano:v2.19.1-python3.8
pipelines:
  pull-requests:
    '**':
      # Validate meltano install command
      - step:
          name: Meltano Install
          script:
            - pip install meltano==2.19.1
            - pip install -r requirements.txt
            - meltano install

      # Testing if every pipeline is working as expected
      - step: 
          name: Testing All the Pipelines
          script: 
            - export START_DATE_TAPGA4_DEV=$(date +%Y-%m-%d -d '1 day ago')
            - export START_DATE_TAPOKTA_DEV=$(date +%Y-%m-%d -d '1 day ago')
            - export START_DATE_TAPGSC_DEV=$(date +%Y-%m-%d -d '1 day ago')
            - meltano --environment=dev run ci-pipeline

      # Building docker image
      - step:
          name: Build Docker File
          services:
            - docker
          script:
            - docker build -t meltano_test .
```

This would be the flow of the continuous integration check.

![CI Pipeline](https://i.postimg.cc/sfBj3H2r/CI-Pipeline.png)
