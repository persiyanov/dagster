Airline Demo
------------

This repository is intended to provide a fleshed-out demo of Dagster and Dagit capabilities. It defines two
realistic data pipelines corresponding to download/ingest and analysis phases of
typical data science workflows, using real-world airline data. Although the view of the pipelines
provided by the Dagster tooling is unified, in typical practice we expect that each pipeline is
likely to be the responsibility of individuals with more or less clearly distinguished roles.

Use the airline demo to familiarize yourself with the features of Dagster in a more
fleshed-out context than the introductory tutorial, and as a reference when building your own
first production pipelines in the system. Comments and suggestions are enthusiastically encouraged.

Getting Started
^^^^^^^^^^^^^^^

To run the airline demo pipelines locally, you'll need:

- To be running python 3.5 or greater (the airline demo uses python 3 type annotations)
- AWS credentials in the ordinary `boto3 credential chain <https://boto3.amazonaws.com/v1/documentation/api/latest/guide/configuration.html>`_.
- A running Postgres database available at ``postgresql://test:test@127.0.0.1:5432/test``. (A
  docker-compose file is provided in this repo at ``dagster/examples/``).


To get up and running:

.. code-block:: console

    # Clone Dagster
    git clone git@github.com:dagster-io/dagster.git
    cd dagster/examples

    # Install all dependencies
    pip install -e .[full]

    # Start a local PostgreSQL database
    docker-compose up -d

    # Load the airline demo in Dagit
    cd dagster_examples/airline_demo
    dagit


You can now view the airline demo in Dagit at `http://127.0.0.1:3000/ <http://127.0.0.1:3000/>`_.


Pipelines & Config
^^^^^^^^^^^^^^^^^^

The demo defines a single repository with two pipelines, in ``airline_demo/pipelines.py``:

- **airline\_demo\_ingest\_pipeline**: This pipeline grabs data archives from S3 and unzips them,
  reads the raw data into Spark, performs some typical manipulations on the data, and then loads
  tables into a data warehouse.
- **airline\_demo\_warehouse\_pipeline**: This pipeline performs typical in-warehouse analysis and
  manipulations using SQL, and then generates and archives analysis artifacts and plots using
  Jupyter notebooks.

Default configuration is provided for both of these pipelines using
`Presets <https://dagster.readthedocs.io/en/stable/sections/api/apidocs/pipeline.html>`_, and the
corresponding YAML files can be found under the ``environments/`` folder.


The Ingest Pipeline
^^^^^^^^^^^^^^^^^^^

.. figure:: https://user-images.githubusercontent.com/609349/62327985-c5d14180-b466-11e9-9e9f-e714f23ceeb0.png
    :align: center
    :alt: Ingest Pipeline


The ``airline_demo_ingest_pipeline`` models the first stage of most project-oriented data science
workflows, in which raw data is consumed from a variety of sources.

For demo purposes, we've put our source files in a publicly-readable S3 bucket. In practice,
these might be files in S3 or other cloud storage systems; publicly available datasets downloaded
over http; or batch files in an SFTP drop.

Running the pipeline locally with test config
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

You can run this pipeline in Dagit by selecting the "Execute" tab, clicking the preset-load button on the top right of
the editor, selecting the ``local_fast`` preset, and then selecting "Start Execution":


.. figure:: https://user-images.githubusercontent.com/609349/62328360-cf0ede00-b467-11e9-883e-dfcceed8687b.png
    :align: center
    :alt: Load Preset & Execute


**Note**: local pipeline execution requires a running PostgreSQL database via the ``docker-compose``
command shown above.

This preset configures the pipeline to consume cut-down versions of the original data files on S3.
It can be good practice to maintain similar test sets so that you can run fast versions of your data
pipelines locally or in test environments. While there are many faults that will not be caught by
using small cut-down or synthetic data sets—for example, data issues that may only appear in
production data or at scale—this practice will allow you to verify the integrity of your pipeline
construction and to catch at least some semantic issues.

The pipeline should now execute and complete in the "Runs" tab of Dagit:


.. figure:: https://user-images.githubusercontent.com/609349/62328637-8572c300-b468-11e9-977e-b67ec4a5eaff.png
    :align: center
    :alt: Pipeline Execution


Reusable Components in Dagster
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Dagster heavily emphasizes building reusable, composable pipelines, and we rely on that
functionality in several places throughout this pipeline.

Let's start by looking at the pipeline definition (in ``airline_demo/pipelines.py``); you'll notice
that we use solid aliasing to reuse the ``s3_to_df`` solid for several ingest steps:


.. literalinclude:: ../../../examples/dagster_examples/airline_demo/pipelines.py
   :lines: 99-106


In general, you won't want every data science user in your organization to have to roll their own
implementation of common functionality like downloading and unzipping files. Instead, you'll want to
abstract common functionality into reusable solids, separating task-specific parameters out into
declarative config, and building up a library of building blocks for new data pipelines. Here,
that's been accomplished by implementing the shared functionality in a ``s3_to_df`` solid and then
re-using that for ingesting several datasets.

Digging deeper, you'll notice that these solids in Dagit are shown with an "Expand" button:


.. figure:: https://user-images.githubusercontent.com/609349/62329597-442fe280-b46b-11e9-9b56-39401bbae50e.png
    :align: center
    :alt: Expand


That's because these are actually `Composite
Solids <https://dagster.readthedocs.io/en/stable/sections/reference/reference.html#composite-solids>`_:
solids that contain other solids. Dagster supports arbitrarily nested, composed solid hierarchies,
which can be really useful for encapsulating functionality as we've done here. Click Expand and
you'll see the solids contained within ``s3_to_df``:


.. figure:: https://user-images.githubusercontent.com/609349/62330149-a806db00-b46c-11e9-8404-0667fa5524c8.png
    :align: center
    :alt: Composite Solid


You can also names and types of all the inputs/outputs of this composite solid and how they are
connected to the inputs/outputs of the solids it contains.

Strongly typed config and outputs
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

The config for each of our solids specifies everything it needs in order to interact with the
external environment. In YAML, an entry in the config for one of our solids aliasing
``s3_to_df`` looks like this:


.. literalinclude:: ../../../examples/dagster_examples/airline_demo/environments/local_fast_ingest.yaml
   :lines: 32-38


Because each of these values is strongly typed, we'll get rich error information in the dagit
config editor (or when running pipelines from the command line) when a config value is incorrectly
specified:

.. figure:: https://user-images.githubusercontent.com/609349/62331551-18176000-b471-11e9-9233-760e3bbea609.png
    :align: center
    :alt: Error


While this may seem like a mere convenience, in production contexts it can dramatically reduce
avoidable errors. Consider boto3's `S3.Client.put_object() <https://boto3.amazonaws.com/v1/documentation/api/latest/reference/services/s3.html#S3.Client.put_object>`_
method, which has 28 parameters, many restricted in obscure ways (for example,
``ServerSideEncryption`` must be one of ``'AES256'|'aws:kms'``). Strongly typed config schemas can catch
any error of this type.

By setting the ``description`` on each of our config members, we also get easily navigable
documentation in dagit. Users of library solids no longer need to investigate implementations in
order to understand what values are required, or what they do—enabling more confident reuse.

.. figure:: https://user-images.githubusercontent.com/609349/62331694-86f4b900-b471-11e9-9702-0b117219f7e6.png
    :align: center
    :alt: Download pipeline config docs


Using Modes for Production Data
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Don't worry, we've got plenty of big(gish) data to run through this pipeline. Instead of the
``local_fast_ingest.yaml`` config fragment, use ``local_full_ingest.yaml``—but be prepared to wait.
You can use this pattern to run your Dagster pipelines against synthetic, anonymized, or subsampled
datasets in test and development environments.

In practice, you'll want to run your pipelines in environments that vary widely in their available
facilities. For example, when running an ingestion pipeline locally for test or development, you
may want to use a local Spark cluster; but when running in production, you will probably want to
target something heftier, like an ephemeral EMR cluster or your organization's on-prem Spark
cluster. Or, you may want to target a locally-running Postgres as your "data warehouse" in local
test/dev, but target a Redshift cluster in production and production tests. Ideally, we want this
to be a matter of flipping a config switch.

Dagster decouples the instantiation of external resources like these from the business logic of your
data pipelines. In Dagster, a set of resources configured for a particular environment is called a
`Mode <https://dagster.readthedocs.io/en/latest/sections/api/apidocs/pipeline.html#modes>`_.

Let's look at how we make configurable resources available to our pipelines with modes. In
``airline_demo/pipelines.py``, you'll find that we define multiple modes within which our pipelines
may run:

.. code-block:: python

    @pipeline(
        # ordered so the local is first and therefore the default
        mode_defs=[local_mode, test_mode, prod_mode],
        ...
    )


This is intended to mimic a typical setup where you may have pipelines running locally on developer
machines, often with a (anonymized or scrubbed) subset of data and with limited compute resources;
remotely in CI/CD, with access to a production or replica environment, but where speed is of the
essence; and remotely in production on live data.

.. literalinclude:: ../../../examples/dagster_examples/airline_demo/pipelines.py
   :lines: 49-72

Here we've defined a ``db_info`` resource that exposes a unified API to our solid logic, but that
wraps two very different underlying assets—in one case, a Postgres database, and in the other,
a Redshift cluster. Let's look more closely at how this is implemented for the Postgres case.

First, we define the ``db_info`` resource itself:

.. code-block:: python

    DbInfo = namedtuple('DbInfo', 'engine url jdbc_url dialect load_table')


This resource exposes a SQLAlchemy engine, the URL of the database (in two forms), metadata about
the SQL dialect that the database speaks, and a utility function, ``load_table``, which loads a
Spark data frame into the target database. In practice, we would probably find that over time we
wanted to add or subtract from this interface, rework its implementations, or factor it into
multiple resources. Because the type definition, config definitions, and implementations are
centralized—rather than spread out across the internals of many solids—this is a relatively
easy task.

Next, we define the config required to instantiate our resource:

.. code-block:: python

    PostgresConfigData = Dict(
        {
            'postgres_username': Field(String),
            'postgres_password': Field(String),
            'postgres_hostname': Field(String),
            'postgres_db_name': Field(String),
        }
    )

Obviously, this config will differ for Redshift, as it might if we had to reach our database through
a proxy server, or using a different authentication schema.

Finally, we bring it all together in the ``postgres_db_info_resource``:

.. literalinclude:: ../../../examples/dagster_examples/airline_demo/resources.py
   :lines: 74-113

By providing strongly typed configuration fields to the ``@resource`` decorator, we now have typeahead
support in dagit and rich error messages for the configuration of our external resources. This can
be extremely valuable in the case of notoriously complex configuration formats, such as Spark's.

Note that Dagit is aware of all modes defined for your pipelines, and will present these to you
(along with the resources they provide) in the pipeline view sidebar:

.. figure:: https://user-images.githubusercontent.com/609349/62332217-7e9d7d80-b473-11e9-8eb7-13198d44eec6.png
    :align: center
    :alt: Modes

Ingesting data to Spark data frames
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Returning to our ``s3_to_df`` solid, note the type signature of these solids.


.. figure:: https://user-images.githubusercontent.com/609349/62332521-e56f6680-b474-11e9-80e7-1b977b8c5ac5.png
    :align: center
    :alt: Ingest pipeline type signature


The output has type ``DataFrame``, a Dagster type that wraps a ``pyspark.sql.DataFrame``.
Defining types of this kind is straightforward and provides additional safety when your solids pass
Python objects to each other:

.. code-block:: python

    DataFrame = as_dagster_type(
        NativeSparkDataFrame, name='DataFrame', description='A Pyspark data frame.'
    )


The transformation solids that follow all use the ``DataFrame`` for their intermediate results.
You might also build DAGs where Pandas data frames, or some other in-memory Python object, are the
common intermediate representation.

Loading data to the warehouse
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

The terminal nodes of this pipeline are all aliased instances of
``load_data_to_database_from_spark``:

.. literalinclude:: ../../../examples/dagster_examples/airline_demo/solids.py
   :lines: 200-218

which abstracts the operation of loading a Spark data frame to a database—either our production
Redshift cluster or our local Postgres in test.

Note how using the ``db_info`` resource simplifies this operation. There's no need to pollute the
implementation of our DAGs with specifics about how to connect to outside databases, credentials,
formatting details, retry or batching logic, etc. This greatly reduces the opportunities for
implementations of these core external operations to drift and introduce subtle bugs, and cleanly
separates infrastructure concerns from the logic of any particular data processing pipeline.

The Warehouse Pipeline
^^^^^^^^^^^^^^^^^^^^^^

.. figure:: https://user-images.githubusercontent.com/609349/62332824-f076c680-b475-11e9-9a58-646401b145f5.png
    :align: center
    :alt: Warehouse Pipeline


The ``airline_demo_warehouse_pipeline`` models the analytics stage of a typical data science workflow.
This is a heterogeneous-by-design process in which analysts, data scientists, and ML engineers
incrementally derive and formalize insights and analytic products (charts, data frames, models, and
metrics) using a wide range of tools—from SQL run directly against the warehouse to Jupyter
notebooks in Python or R.

The ``sql_solid``: wrapping foreign code in a solid
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

How do we actually package the SQL for execution and display with Dagster and Dagit? We've built a
function, ``sql_solid``, that is designed to take a SQL query and return a solid.

Fundamentally, all this is doing is invoking ``execute`` on the ``db_info.engine`` provided by the
context:

.. code-block:: python

    context.resources.db_info.engine.execute(text(sql_statement))


.. figure:: https://user-images.githubusercontent.com/609349/62580646-93a35380-b85b-11e9-8118-8c4058ebb55d.png
    :align: center
    :alt: Warehouse Pipeline SQL


Also, note how straightforward the interface to ``sql_solid`` is. To define a new solid executing a
relatively complex computation against the data warehouse, an analyst only needs to supply light
metadata along with their SQL query:


.. literalinclude:: ../../../examples/dagster_examples/airline_demo/solids.py
   :lines: 344-382


This kind of interface can supercharge the work of analysts who are highly skilled in SQL, but for
whom fully general-purpose programming in Python may be uncomfortable. Analysts need only master a
very constrained interface in order to define their SQL statements' data dependencies and outputs in
a way that allows them to be orchestrated and explored in heterogeneous DAGs containing other forms
of computation, and must make only minimal changes to code (in some cases, no changes) in order to
take advantage of centralized infrastructure work.

Dagstermill: executing and checkpointing notebooks
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Notebooks are a flexible way and increasingly ubiquitous way to explore datasets and build data
products, from transformed or augmented datasets to charts and metrics to deployable machine
learning models. But their very flexibility—the way they allow analysts to move seamlessly from
interactive exploration of data to scratch code and finally to actionable results—can make them
difficult to productionize.

Our approach, built on `papermill <https://github.com/nteract/papermill>`_, embraces the diversity
of code in notebooks. Rather than requiring notebooks to be rewritten or "cleaned up" in order to
consume inputs from other nodes in a pipeline or to produce outputs for downstream nodes, we can
just add a few cells to provide a thin wrapper around an existing notebook.

Notebook solids can be identified by the small red ``ipynb`` tag in Dagit. As with SQL solids, they
can be explored from within Dagit by drilling down on the tag:


.. figure:: https://user-images.githubusercontent.com/609349/62580619-80908380-b85b-11e9-9d6d-c52796da8ffd.png
    :align: center
    :alt: Warehouse Pipeline Notebook


Let's start with the definition of our ``notebook_solid`` helper:


.. literalinclude:: ../../../examples/dagster_examples/airline_demo/solids.py
   :lines: 40-41


This is just a wrapper around Dagstermill's ``define_dagstermill_solid`` which tells Dagstermill
where to look for the notebooks. We define a new solid by using this function and referencing
a notebook file:


.. literalinclude:: ../../../examples/dagster_examples/airline_demo/solids.py
   :lines: 440-462


As usual, we define the inputs and outputs of the new solid. Within the notebook itself, we only
need to add a single line as the first cell:

.. code-block:: python

    import dagstermill


Then, in a cell with the ``parameters`` tag, we write:

.. code-block:: python

    context = dagstermill.get_context()

    eastbound_delays = 'eastbound_delays'
    westbound_delays = 'westbound_delays'


When dagstermill executes this notebook as part of a pipeline, values for these inputs will be
injected from the output of upstream solids.

Finally, at the very end of the notebook, we yield the result back to the pipeline:

.. code-block:: python

    dagstermill.yield_result(LocalFileHandle(pdf_path))
