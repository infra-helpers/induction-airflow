Induction to Workflow Engines - Airflow
=======================================

[![Canyon, by Patrick Hendry on Unsplash](img/patrick-hendry-Nijc2oQz10I-unsplash-small.jpg)](https://unsplash.com/photos/Nijc2oQz10I)

# Overview
[That repository](https://github.com/infra-helpers/induction-airflow)
aims at providing end-to-end examples introducing how to setup, run and
monitor workflows, mainly for data processing pipelines.

In those tutorials, Airflow is used. A full end-to-end example is explained
step by step, and actually used for the [Open Travel Data (OPTD)
project](https://github.com/opentraveldata/opentraveldata).

The full details on how to setup an Airflow server on Proxmox LXC containers
are given in the [dedicated `airflow/` sub-folder](airflow/).
Such an Airflow cluster is actually the orchestration engine of the
[Open Travel Data (OPTD)
project](https://github.com/opentraveldata/opentraveldata).

For convenience, most of the Airflow examples are demonstrated both on a local
single-node installation (_e.g._, on a laptop) and on on the above-mentioned
cluster.

## Endpoints
* Airflow:
  + Local server: http://localhost:8080
  + Remote server: https://airflow.example.com

# Table of Content (ToC)
- [Induction to Workflow Engines - Airflow](#induction-to-workflow-engines---airflow)
- [Overview](#overview)
  * [Endpoints](#endpoints)
- [Table of Content (ToC)](#table-of-content--toc-)
- [References](#references)
- [Configuration](#configuration)

<small><i><a href='http://ecotrust-canada.github.io/markdown-toc/'>Table of contents generated with markdown-toc</a></i></small>


# References
* [Setup of a Proxmox host](https://github.com/cloud-helpers/kubernetes-hard-way-bare-metal/blob/master/proxmox/README.md)
* [Setup of LXC containers on Proxmox](https://github.com/cloud-helpers/kubernetes-hard-way-bare-metal/blob/master/lxc/README.md)
* [Induction to Monitoring with Elasticsearch (ES), Kibana and Kafka](https://github.com/infra-helpers/induction-monitoring)

# Configuration
* See [`airflow/`](airflow/)

