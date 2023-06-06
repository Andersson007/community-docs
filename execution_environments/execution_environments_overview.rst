.. _ee_overview:

***************************************
Ansible Execution Environments Overview
***************************************

.. contents::
   :local:

.. _terminology:

Terminology
===========

Notions used in this document:

* **ansible-core**: you install it using ``pip install ansible-core``; includes command-line tools such as ``ansible-playbook`` and ``ansible-galaxy``, the Ansible language and a set of `builtin modules and plugins <https://docs.ansible.com/ansible/latest/collections/ansible/builtin/index.html>`_.
* **Ansible collections**: a format in which Ansible content is distributed that can contain playbooks, roles, modules and plugins.
* **Ansible package**: you install it using ``pip install ansible`` or an OS distribution package manager; it provides ansible-core and a big set of Ansible collections in the *batteries included* manner.
* **Ansible**: in the context of this document, is the Ansible package or ``ansible-core`` plus a set of collections installed on the Ansible control node.
* **Ansible control node**: the machine from which you run Ansible.
* **Container Runtime**: it is what you typically use ``podman`` or ``docker`` for running containers.

.. _ee_rationale:

Why Execution Environments were introduced
==========================================

The Ansible Execution Environments were introduced to resolve the following issues
and provide all benefits you can get from using containerization.

Dependencies
------------

Software applications typically have dependencies.
It can be software libraries, configuration files or other services, to name a few, and Ansible,
being an exceptional automation tool, is not an exception in terms of the mentioned.

Traditionally, application dependencies are installed by administrators on top of
an operating system using packaging management tools like RPM or Python-pip.

The major drawback of such approach is that an application might require versions
of dependencies different from those provided by default.

In case of Ansible, a typical installation consists of ansible-core and a set of Ansible collections.

At present, there are more than a hundred collections included in the Ansible package and
hundreds more are available on Ansible Galaxy and Automation Hub for manual installation.
Many of them have dependencies for their plugins, modules, roles and playbooks they provide.

The Ansible collections can depend on the following pieces of software and their versions:

* ansible-core 
* Python
* Python packages
* system packages
* other Ansible collections

The dependencies have to be installed and sometimes can conflict with each other.

One way to **partly** resolve dependency issue is
to use Python virtual environments on Ansible control node.
However, applied to Ansible, it has drawbacks and natural limitations.

Content separation
------------------

Typically users do not really need the whole Ansible package with that hundred collections.
In most cases, and it is considered a good practice, you need to install only ``ansible-core``
plus a set of collections for specific tasks, nothing more.

Let's say, there is an Ansible control node or a tool like Ansible AWX/Controller used by several users.

Each user works with a limited set of services and wants to use only ``ansible-core``
and a corresponding set of Ansible collections automating those services.

Each user is not interested in having someone else's content the users do not need including dependencies
which can potentially brake their pipelines.
They might also want to preserve their installation clean from unnecessary packages
and files belonging their colleagues.

Simply put, users might want their content be separated.

Portability
-----------

An Ansible user writes content for Ansible locally and wants to leverage the container technology
to make your automation runtimes portable, shareable and easily deployable to testing and production environments.

What is Execution Environment
=============================

Ansible, as a software application, can run in a container, thus, it can benefit from containerization the same as most other applications.

The Ansible automation execution environment (hereinafter, execution environment or EE) is a container image serving as Ansible control node.

The EE image contains:

* ansible-core
* ansible-runner
* none or more Ansible collections
* Python
* Python and system dependencies
* custom user needs

Tools you can use EEs with
-------------------------

You can use EEs with:

* ansible-navigator
* `Ansible AWX <https://docs.ansible.com/automation-controller/latest/html/userguide/execution_environments.html#use-an-execution-environment-in-jobs>`_
* `Automation controller <https://docs.ansible.com/automation-controller/latest/html/userguide/execution_environments.html#use-an-execution-environment-in-jobs>`_
* ansible-runner
* VS Code `Ansible <https://marketplace.visualstudio.com/items?itemName=redhat.ansible>`_ and `Dev Containers <https://code.visualstudio.com/docs/devcontainers/containers>`_ extensions

TODO: Add links for the items above after we determine their permanent places to live (and move if needed)

.. _how_to_prepare_environment:

How to prepare your environment to build and test EEs
=====================================================

Install the following packages:

* podman or docker
* python-pip: to install the tools
* ansible-builder: to build EEs
* ansible-navigator: to run EEs

On distributions using DNF as a package manager:

.. code-block:: bash

  $ sudo dnf install -y podman python3-pip

.. code-block:: bash

  $ pip3 install ansible-builder ansible-navigator


What to read next
=================

TODO: paste links to docs when written

To read next:

* `How to build and test EE <ADD LINK WHEN WRITTEN>`_ guide
* `Ansible Builder documentation <https://ansible-builder.readthedocs.io/en/stable/>`_
* `Ansible Navigator EE-specific overview <ADD LINK WHEN WRITTEN>`_
