.. _partial-parsing:

Partial parsing
===============

Starting in the 1.4 version, Cosmos tries to leverage dbt's partial parsing (``partial_parse.msgpack``) to speed up both the task execution and the DAG parsing (if using ``LoadMode.DBT_LS``).

This feature is bound to `dbt partial parsing limitations <https://docs.getdbt.com/reference/parsing#known-limitations>`_.
As an example, ``dbt`` requires the same ``--vars``, ``--target``, ``--profile``, and ``profile.yml`` environment variables (as called by the ``env_var()`` macro) while running dbt commands, otherwise it will reparse the project from scratch.

.. _partial-parsing-profile-configuration:

Profile configuration
~~~~~~~~~~~~~~~~~~~~~

To respect the dbt requirement of having the same profile to benefit from partial parsing, Cosmos users should either:

- If using Cosmos profile mapping (``ProfileConfig(profile_mapping=...``), disable using mocked profile mappings by setting ``render_config=RenderConfig(enable_mock_profile=False)``
- Declare their own ``profiles.yml`` file, via ``ProfileConfig(profiles_yml_filepath=...)``

If users don't follow these guidelines, Cosmos will use different profiles to parse the dbt project and to run tasks, and the user won't leverage dbt partial parsing.
Their logs will contain multiple ``INFO`` messages similar to the following, meaning that Cosmos is not using partial parsing:

.. code-block:: text

    13:33:16  Unable to do partial parsing because profile has changed
    13:33:16  Unable to do partial parsing because env vars used in profiles.yml have changed

dbt vars
~~~~~~~~

If the `Apache Airflow® <https://airflow.apache.org/>`_ scheduler and worker processes run in the same node, users must ensure the dbt ``--vars`` flag is the same in the ``RenderConfig`` and ``ExecutionConfig``.

Otherwise, users may see messages similar to the following in their logs:

.. code-block:: text

    [2024-03-14, 17:04:57 GMT] {{subprocess.py:94}} INFO - Unable to do partial parsing because config vars, config profile, or config target have changed


Caching
~~~~~~~

If the dbt project ``target`` directory has a ``partial_parse.msgpack``, Cosmos will attempt to use it.

There is a chance, however, that the file is stale or was generated in a way that is different to how Cosmos runs the dbt commands.

Therefore, Cosmos also caches the most up-to-date ``partial_parse.msgpack`` file after running a dbt command in the `system temporary directory <https://docs.python.org/3/library/tempfile.html#tempfile.gettempdir>`_.
With this, unless there are code changes, each Airflow node should only run the dbt command with a full dbt project parse once, and benefit from partial parsing from then onwards.


Caching is enabled by default.
It is possible to disable caching or override the directory that Cosmos uses caching with the Airflow configuration:

.. code-block:: cfg

    [cosmos]
    cache_dir = path/to/docs/here  # to override default caching directory (by default, uses the system temporary directory)
    enable_cache_partial_parse = False  # to disable caching (enabled by default)

Or environment variable:

.. code-block:: cfg

    AIRFLOW__COSMOS__CACHE_DIR="path/to/docs/here"  # to override default caching directory (by default, uses the system temporary directory)
    AIRFLOW__COSMOS__ENABLE_CACHE_PARTIAL_PARSE="False"  # to disable caching (enabled by default)

Because this cache is local to each worker node, it does not survive worker replacement: with ephemeral or
auto-scaling workers (e.g. MWAA, Astro, Kubernetes), each new worker pays the cost of a full dbt parse on its first
task. Starting with Cosmos 1.16, if a remote cache directory is configured (see :ref:`remote_cache_dir`), users can
opt in to sharing the partial parse cache across workers:

.. code-block:: cfg

    [cosmos]
    remote_cache_dir = s3://your_s3_bucket/cache_dir/
    enable_remote_cache_partial_parse = True

With this setting, a worker whose local cache is still empty downloads the latest ``partial_parse.msgpack`` from the
remote cache directory before running the dbt command, and tasks upload the latest artifacts back to the remote cache
after a successful run (the upload is skipped when the content did not change).

Learn more about :doc:`caching </optimize_performance/caching>` and :doc:`Cosmos Airflow configurations </reference/configs/cosmos-conf>`.

Disabling
~~~~~~~~~

To switch off partial parsing in Cosmos, use the argument ``partial_parse=False`` in the ``ProjectConfig``.
