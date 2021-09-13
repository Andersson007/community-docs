*************************************
Quick-start integration testing guide
*************************************

This guide describes all steps needed to add integration tests for your changes to a collection and run them locally using the ``ansible-test`` command.

.. contents:: Topics

Basics
======

.. note::

  If you find any inconsistencies or places in this document that can be improved / clarified, please raise an issue or pull request to fix it.

.. note::

  Some collections do not have integration tests.

.. note::

  If there are any difficulties with writing / running integration tests or you are not sure if the case can be covered, feel free to submit your pull request without the tests. Other contributors can help you with them later if needed.

This section provides only brief explanation of what integration tests are. You will learn details and will see examples in the following sections.

Integration tests are functional tests of modules and plugins. We will use the word ``module`` throughout the document implying both modules and plugins.

With this kind of tests, we check if a module as a whole satisfies its functional requirements. Simply put, we check that features work as expected and users will get the outcome described in the module's documentation.

We check modules with playbooks that invoke those modules. We pass standalone parameters and their combinations, and check what the module reports with the `assert <https://docs.ansible.com/ansible/latest/collections/ansible/builtin/assert_module.html>`_ module and the actual state of the system after each task.

Here is an example.

Let's say we want to test the ``postgresql_user`` module invoked with the ``name`` option. We expect that the module will create a user that we are passing through the ``name`` option and will report that the system state has changed. We cannot rely on only what the module reports. To be sure that the user has been actually created, we will query our database with another module to see if the user exists.

.. code:: yaml

  - name: Create PostgreSQL user and store module's output to the result variable
    postgresql_user:
      name: test_user
    register: result

  - name: Check the module returns what we expect
    assert:
      that:
        - result is changed

  - name: Check actual system state with another module, in other words, that the user exists
    postgresql_query:
      query: SELECT * FROM pg_authid WHERE rolename = 'test_user'
    register: query_result

  - name: We expect it returns one row, check it
    assert:
      that:
        - query_result.rowcount == 1

You will learn details in the following sections.

The basic entity of Ansible integration tests is a ``target``.

The target is an `Ansible role <https://docs.ansible.com/ansible/latest/user_guide/playbooks_reuse_roles.html>`_ stored in the ``tests/integration/targets`` directory of a collection repository file tree.

The target role contains all needed to test a module.

Names of targets contain a module name they test.

Target names that start with ``setup_`` are usually executed as dependencies before module and plugin targets start execution. We will describe this kind of targets later in detail in the :ref:`Writing tests from scratch<Writing-tests-from-scratch>` section.

To run integration tests, we use the ``ansible-test`` utility shipped with the ``ansible-core`` and ``ansible`` packages. This is described in the :ref:`Run integration tests<Run-integration-tests>` section.

After you finish your integration tests, refert to the `Create a PR quick-start guide <https://github.com/ansible/community-docs/blob/main/create_pr_quick_start_guide.rst>`_ to learn how to submit a pull request.

.. _Prepare-local-environment:

Prepare local environment
=========================

Before starting working on integration tests, to be able to run them locally, you need to prepare your environment.

To learn how to do it quickly, refer to the `Quick-start development guide <https://github.com/ansible/community-docs/blob/main/create_pr_quick_start_guide.rst#prepare-your-environment>`_.

.. _Determine-if-integration-tests-exists:

Determine if integration tests exist
====================================

Provided that integration tests for a collection exist, they are stored in ``tests/integration/targets`` subdirectories in the collection repository.

If you already have your local environment :ref:`prepared<Prepare-local-environment>`, you can run the following command being in the collection's root directory to list all the available targets:

.. code:: bash

  ansible-test integration --list-targets

If you use ``bash`` and the ``argcomplete`` package is installed via ``pip`` on your system, you can also get a full target list by doing: ``ansible-test integration <tab><tab>``.
Alternatively, you can check if the ``tests/integration/targets`` contains a corresponding directory named as the module.

For example, the tests for the ``postgresql_user`` module of the ``community.postgresql`` collection are stored in the ``tests/integration/targets/postgresql_user`` directory of the collection's source tree.

If there is no corresponding target there, it means that the module does not have integration tests. In this case, think of adding integration tests for the module. Refer to the :ref:`Writing tests from scratch<Writing-tests-from-scratch>` section for details.

.. _Adding-tests-to-existing-ones:

Adding your tests to existing ones
==================================

The test tasks are stored in the ``tests/integration/targets/<target_name>/tasks`` directory.

The ``main.yml`` file holds test tasks and includes other test files.
Look for a suitable test file to integrate your tests or create and include / import a separate test file.
You can use one of the existing test files as a draft.

When fixing a bug
-----------------

When fixing a bug:

1. :ref:`Determine if integration tests for the module exist<Determine-if-integration-tests-exist>`. If they do not, refer to the :ref:`Writing tests from scratch <Writing-tests-from-scratch>` section.
2. Add a task which reproduces the to an appropriate file within the ``tests/integration/targets/<target_name>/tasks`` directory.
3. :ref:`Run the tests<Run-integration-tests>`, they should fail.
4. If they do not fail, re-check if your environment / test task satisfies the conditions described in the ``Steps to Reproduce`` section of the issue.
5. If you reproduce the bug and tests fail, change the code. 
6. :ref:`Run the tests<Run-integration-tests>` again.
7. If they fail, repeat steps 5-6 until the tests pass.

Here is an example.

Let's say we got an issue in the ``community.postgresql`` collection. When users pass a name containing underscores to the ``postgresql_user`` module, the module fails.

We cloned the collection repository to the ``~/ansible_collections/community/postgresql`` directory and :ref:`prepared our environment <Prepare-local-environment>`. Being there, we run ``ansible-test integration --list-targets`` and it shows a target called ``postgresql_user``. It means that we already have tests for the module.

We start with reproducing the bug.

First, we look into the ``tests/integration/targets/<target_name>/tasks/main.yml`` file. In case of the ``community.postgresql``, it imports other files from the ``tasks`` directory. We looked through the files - ``postgresql_user_general.yml`` looks like an appropriate one to add our tests.

.. code:: yaml

  # General tests:
  - import_tasks: postgresql_user_general.yml
    when: postgres_version_resp.stdout is version('9.4', '>=')

We will add the following code to the file.

.. code:: yaml

  # https://github.com/ansible-collections/community.postgresql/issues/NUM
  - name: Test user name containing underscore
    postgresql_user:
      name: underscored_user
    register: result

  - name: Check the module returns what we expect
    assert:
      that:
        - result is changed

  - name: Query the database if the user exists
    postgresql_query:
      query: SELECT * FROM pg_authid WHERE rolename = 'underscored_user'
    register: result

  - name: Check the database returns one row
    assert:
      that:
        - query_result.rowcount == 1

When we :ref:`run the tests<Run-integration-tests>` passing ``postgresql_user`` as a test target, this task must fail.

Then we will fix the bug and run the same tests again. If they pass, we will consider the bug fixed and will submit a pull request.

When adding a new feature
-------------------------

.. note::

  The process described in this section is also applicable when a feature already exists but does not have integration tests and you want to cover it.

.. note::

  If you do not implement the feature you want yet, you can start with writing integration tests for it. Of course they will not work as the code does not exist at the moment but it can help you design better implementation before writing the code.

When adding new features, the process of adding tests consists of the following steps:

1. :ref:`Determine if integration tests for the module exists<Determine-if-integration-tests-exist>`. If they do not, refer to the :ref:`Writing tests from scratch<Writing-tests-from-scratch>` section.
2. Find an appropriate file for your tests within the ``tests/integration/targets/<target_name>/tasks`` directory.
3. Cover your option. Refer to the :ref:`Recommendations on coverage<Recommendations-on-coverage>` section for details.
4. :ref:`Run the tests<Run-integration-tests>`.
5. If they fail, see the test output for details. Fix your code or tests and run the tests again.
6. Repeat steps 4-5 until the tests pass.

Here is an example.

Let's say we decided to add a new option called ``add_attribute`` to the ``postgresql_user`` module of the ``community.postgresql`` collection.

The option is boolean. If set to ``yes``, it adds an additional attribute to a database user.

We cloned the collection repository to the ``~/ansible_collections/community/postgresql`` directory and :ref:`prepared our environment<Prepare-local-environment>`. Being there, we run ``ansible-test integration --list-targets`` and it shows a target called ``postgresql_user``. It means that we already have tests for the module.

First, we look into ``tests/integration/targets/<target_name>/tasks/main.yml``. In case of the ``community.postgresql``, it imports other files from the ``tasks`` directory. We looked through the files - ``postgresql_user_general.yml`` looks like an appropriate one to add our tests.

.. code:: yaml

  # General tests:
  - import_tasks: postgresql_user_general.yml
    when: postgres_version_resp.stdout is version('9.4', '>=')

We will add the following code to the file.

.. code:: yaml

  # https://github.com/ansible-collections/community.postgresql/issues/NUM
  # We should also run the same tasks with check_mode: yes. We omit it here for simplicity.
  - name: Test for new_option, create new user WITHOUT the attribute
    postgresql_user:
      name: test_user
      add_attribute: no
    register: result

  - name: Check the module returns what we expect
    assert:
      that:
        - result is changed

  - name: Query the database if the user exists but does not have the attribute (it is NULL)
    postgresql_query:
      query: SELECT * FROM pg_authid WHERE rolename = 'test_user' AND attribute = NULL
    register: result

  - name: Check the database returns one row
    assert:
      that:
        - query_result.rowcount == 1

  - name: Test for new_option, create new user WITH the attribute
    postgresql_user:
      name: test_user
      add_attribute: yes
    register: result

  - name: Check the module returns what we expect
    assert:
      that:
        - result is changed

  - name: Query the database if the user has the attribute (it is TRUE)
    postgresql_query:
      query: SELECT * FROM pg_authid WHERE rolename = 'test_user' AND attribute = 't'
    register: result

  - name: Check the database returns one row
    assert:
      that:
        - query_result.rowcount == 1

When we :ref:`run the tests<Run-integration-tests>` with ``postgresql_user`` passed as a test target.

In real world, we would alternate the tasks above with the same tasks run with the ``check_mode: yes`` option to be sure our option works as expected in check-mode as well. Refer to the :ref:`Recommendations on coverage<Recommendations-on-coverage>` section for details.

If we expect a task to fail, we use the ``ignore_errors: yes`` option and check that the task actually failed and returned the message we expect:

.. code:: yaml

  - name: Test for fail_when_true option
    postgresql_user:
      name: test_user
      fail_when_true: yes
    register: result
    ignore_errors: yes

  - name: Check the module fails and returns message we expect
    assert:
      that:
        - result is failed
        - result.msg == 'The message we expect'

.. _Writing-tests-from-scratch:

Writing tests from scratch
==========================

This section covers cases when:

- There are no integration tests for a collection / group of modules in a collection at all.
- You are adding a new module and you want to cover it.
- You want to cover a module that already exists but integration tests for the module are missed.

In other words, there are currently no tests for a module regardless of whether the module exists or not.

If the module already has tests, refer to the :ref:`Adding test to existing ones<Adding-tests-to-existing-ones>` section.

Abstract example
----------------

Here is a simplified abstract example.

Let's say we are going to cover a new module in the ``community.abstract`` collection which interacts with some service.

We :ref:`checked<Determine-if-integration-tests-exist>` and figure out that there are no integration tests at all.

We should basically do the following:

1. Install and run the service with a ``setup`` target.
2. Create a test target.
3. :ref:`Cover our module with tests<Recommendations-on-coverage>`.
4. :ref:`Run the tests<Run-integration-tests>`.
5. Fix the code / tests if needed, run the tests again, and repeat the cycle until they pass.

You can reuse the ``setup`` target when implementing targets for other modules for the service later.

1. Clone the collection to the ``~/ansble_collections/community.abstract`` directory on your local machine.

2. Being in ``~/ansble_collections/community.abstract``, create directories for the ``setup`` target:

.. code:: bash

  mkdir -p tests/integration/targets/setup_abstract_service/tasks

3. Write all the tasks needed to prepare the environment, install, and run the service.

For simplicity, let's imagine that the service is available in the native distribution repositories and no sophisticated environment configuration is required.

Add the following tasks to the ``tests/integration/targets/setup_abstract_service/tasks/main.yml`` file to install and run the service:

.. code:: yaml

  - name: Install abstract service
    package:
      name: abstract_service

  - name: Run the service
    systemd:
      name: abstract_service
      state: started

This is a very simplified example.

4. Add the target for the module you test.

Let's say the module is called ``abstact_service_info``. Create the following directory structure in the target:

.. code:: bash

  mkdir -p tests/integration/targets/abstract_service_info/tasks
  mkdir -p tests/integration/targets/abstract_service_info/meta

Add all subdirectories needed. For example, if you are going to use defaults and files, add the ``defaults`` and ``files`` directories, and so on. The approach is the same as when you are creating a role.

5. To make the ``setup_abstract_service`` target running before the module's target, add the following lines to the ``tests/integration/targets/abstract_service_info/meta/main.yml`` file.

.. code:: yaml

  dependencies:
    - setup_abstract_service

6. Start with writing a single standalone task to check that your module can interact with the service.

We assume that the ``anstract_service_info`` module fetches some information from the ``abstract_service`` and it has two connection parameters.

Among other fields, it returns a field called ``version`` containing a service version.

Add the following to ``tests/integration/targets/abstract_service_info/tasks/main.yml``:

.. code:: yaml

  - name: Fetch info from abstract service
    anstract_service_info:
      host: 127.0.0.1  # We assume the service accepts local connection by default
      port: 1234       # We assume that the service is listening this port by default
    register: result   # This variable will contain the returned JSON including the server version

  - name: Test the output
    assert:
      that:
        - result.version == '1.0.0'  # Check version field contains what we expect

7. :ref:`Run the tests<Run-integration-tests>` with the ``-vvv`` argument.

If there are any issues with connectivity (for example, the service does not listening / accepting connections or anything else) or with the code, the play will fail.

Examine the output to see at which step the failure occurred. Investigate the reason, fix, and run again. Repeat the cycle until the test passes.

8. If the test succeeds, write more tests. Refer to the :ref:`Recommendations on coverage<Recommendations-on-coverage>` section for details.

Real example
------------

Here is a real example of writing integration tests from scratch for the ``community.postgresql.postgresql_info`` module.

For the sake of simplicity, we will create very basic tests which we will run using the Ubuntu 20.04 test container.

We use ``Linux`` as a work environment and have ``git`` and ``docker`` installed and running.

We also `installed <https://docs.ansible.com/ansible/devel/installation_guide/intro_installation.html>`_ ``ansible-core``.

1. Create the following directories in your home directory:

.. code:: bash

  mkdir -p ~/ansible_collections/community

2. Fork the `collection repository <https://github.com/ansible-collections/community.postgresql>`_ through the GitHub web interface.

3. Clone the forked repository from your profile to the created path:

.. code:: bash

  git clone https://github.com/YOURACC/community.postgresql.git ~/ansible_collections/community/postgresql

If you prefer to use the SSH protocol:

.. code:: bash

  git clone git@github.com:YOURACC/community.postgresql.git ~/ansible_collections/community/postgresql

4. Go to the cloned repository:

.. code:: bash

  cd ~/ansible_collections/community/postgresql

5. Be sure you are in the default branch:

.. code:: bash

  git status

6. Checkout a test branch:

.. code:: bash

  git checkout -b postgresql_info_tests


7. Because we already have tests for the ``postgresql_info`` module, run the following command:

.. code:: bash

  rm -rf tests/integration/targets/*

The state now is like we do not have integration tests for the ``community.postgresql`` collection at all. So we can start writing integration tests from scratch.

8. We will start with creating a ``setup`` target that will install all required packages and will launch PostgreSQL. Create the following directories:

.. code:: bash

  mkdir -p tests/integration/targets/setup_postgresql_db/tasks

9. Create the ``tests/integration/targets/setup_postgresql_db/tasks/main.yml`` file and add the following tasks to it:

.. code:: yaml

  - name: Install required packages
    package:
      name:
        - apt-utils
        - postgresql
        - postgresql-common
        - python3-psycopg2

  - name: Initialize PostgreSQL
    shell: . /usr/share/postgresql-common/maintscripts-functions && set_system_locale && /usr/bin/pg_createcluster -u postgres 12 main
    args:
      creates: /etc/postgresql/12/

  - name: Start PostgreSQL service
    service:
      name: postgresql
      state: started

That is enough for our very basic example.

10. Then, create the following directories for the ``postgresql_info`` target:

.. code:: bash

  mkdir -p tests/integration/targets/postgresql_info/tasks tests/integration/targets/postgresql_info/meta

11. To make the ``setup_postgresql_db`` target running before the ``postgresql_info`` target as a dependency, create the ``tests/integration/targets/postgresql_info/meta/main.yml`` file and add the following code to it:

.. code:: yaml

  dependencies:
    - setup_postgresql_db

12. Now we are ready to add our first test task for the ``postgresql_info`` module. Create the ``tests/integration/targets/postgresql_info/tasks/main.yml`` file and add the following code to it:

.. code:: yaml

  - name: Test postgresql_info module
    become: yes
    become_user: postgres
    postgresql_info:
      login_user: postgres
      login_db: postgres
    register: result

  - name: Check the module returns what we expect
    assert:
      that:
        - result is not changed
        - result.version.major == 12
        - result.version.minor == 8

In the first task, we run the ``postgresql_info`` module to fetch information from the database we installed and launched with the ``setup_postgresql_db`` target. We are saving values returned by the module into the ``result`` variable.

In the second task, we check with the ``assert`` module what our task returns. We expect that, among other things, it returns the server version and reports that the system state has not been changed.

13. Run the tests in the Ubuntu 20.04 docker container:

.. code:: bash

  ansible-test integration postgresql_info --docker ubuntu2004 -vvv

The tests should pass. If we look at the output, we should see something like the following:

.. code:: bash

  TASK [postgresql_info : Check the module returns what we expect] ***************
  ok: [testhost] => {
    "changed": false,
    "msg": "All assertions passed"
  }

If your tests fail when you are working on your project, examine the output to see at which step the failure occurred. Investigate the reason, fix, and run again. Repeat the cycle until the test passes. If the test succeeds, write more tests. Refer to the :ref:`Recommendations on coverage<Recommendations-on-coverage>` section for details.

.. _Recommendations-on-coverage:

Recommendations on coverage
===========================

Bugfixes
--------

Before fixing code, create a test case in an :ref:`appropriate test target<Determine-if-integration-tests-exist>` reproducing the bug provided by the issue reporter and described in the ``Steps to Reproduce`` issue section. :ref:`Run<Run-integration-tests>` the tests.

If you failed to reproduce the bug, ask the reporter to provide additional information. Maybe the cause is just wrong environment settings.

In very environment specific cases that cannot be reproduced in integration tests, manual testing by issue reporter or other interested users is required.

Refactoring code
----------------

When refactoring code, always check that related options are covered in a :ref:`corresponding test target<Determine-if-integration-tests-exist>`. Do not assume if the test target exists, everything is (well) covered.

For more information on how features should be tested, refer to :ref:`this section<Covering-modules-new-features>`. 

.. _Covering-modules-new-features:

Covering modules / new features
-------------------------------

When covering a module, cover all its options separately and their meaningful combinations. Every possible use of the module should be tested against:

- Idepmotency (Does re-running a task report no changes?)
- Check-mode (Does dry-running a task behaves the same as a real run? Does it not make any changes?)
- Return values (Does the module return values consistently under different conditions?)

Each of test actions will have to be tested at least six times:

- Perform an action in check-mode if supported (this should indicate a change).
- Check with another module that the changes have ``not`` been actually made.
- Perform the action for real (this should indicate a change).
- Check with another module that the changes have been actually made.
- Perform the action again in check-mode (this should indicate ``no`` change).
- Perform the action again for real (this should indicate ``no`` change).

To check a task:

- Register the outcome of the task as a variable, for example, ``register: result``. Using the `assert <https://docs.ansible.com/ansible/latest/collections/ansible/builtin/assert_module.html>`_ module, check:

  + If ``- result is changed`` or not.
  + Expected return values.
- If the module changes a system, check the actual system state using other modules. For example, if the module changes a file, we can check that the file has been changed by checking its checksum with the ``ansible.builtin.stat`` module before and after the test tasks.
- Run the same task with ``check_mode: yes`` (if check-mode is supported by the module). Check with other modules that the actual system state has not been changed.
- Cover cases when the module must fail. Use the ``ignore_errors: yes`` option and check the returned message with the ``assert`` module.

Example:

.. code:: yaml

  - name: Task to fail
    abstract_module:
        ...
    register: result
    ignore_errors: yes

  - name: Check the task fails and its error message
    assert:
      that:
        - result is failed
        - result.msg == 'Message we expect'

Here is a summary:

- Cover options and their sensible combinations.
- Check returned values.
- Cover check-mode if supported.
- Check a system state using other modules.
- Check when a module must fail and error messages.

.. _Run-integration-tests:

Run integration tests
=====================

In the following examples, we will use ``Docker`` to run integration tests locally.

Be sure, you :ref:`prepared your local environment<Prepare-local-environment>` first.

We assume that you are in the ``~/ansible_collections/NAMESPACE/COLLECTION`` directory.

After you change the tests, you can run them with the following command:

.. code:: bash

  ansible-test integration <target_name> --docker <distro>

The ``target_name`` is a test role directory containing the tests. For example, if the test files you changed are stored in the ``tests/integration/targets/postgresql_info/`` directory, the command will be:

.. code:: bash

  ansible-test integration postgresql_info --docker fedora34

You can use the ``-vv`` or ``-vvv`` argument if you need more detailed output.

In the examples above, the ``fedora34`` test image will be automatically downloaded and used to create and run a test container.

See the `list of supported container images <https://docs.ansible.com/ansible/latest/dev_guide/testing_integration.html#container-images>`_.

In some cases, for example, for platform independent tests, the ``default`` test image is required. Use the ``--docker default`` or just ``--docker`` option without specifying a distribution in this case.

If you are not sure which image you should use, ask collection maintainers for clarification.

For details about running integration tests with ``Docker``, refer to the `Ansible documentation <https://docs.ansible.com/ansible/latest/dev_guide/testing_integration.html#tests-in-docker-containers>`_.

Going deeper
============

To scrutinize the topic, refer to the following documents:

- `Testing guide <https://docs.ansible.com/ansible/latest/dev_guide/testing.html>`_.
- `Integration tests guide <https://docs.ansible.com/ansible/latest/dev_guide/testing_integration.html>`_.
- `Testing collections <https://docs.ansible.com/ansible/latest/dev_guide/developing_collections_testing.html#testing-collections>`_.
- `Resource module integration tests <https://docs.ansible.com/ansible/latest/network/dev_guide/developing_resource_modules_network.html#resource-module-integration-tests>`_.
- `How to test a pull request locally <https://github.com/ansible/community-docs/blob/main/test_pr_locally_guide.rst>`_.
