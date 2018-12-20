# Dependency Monkey Design Document

This document discusses Dependency Monkey design in project Thoth. Dependency
Monkey is a tool which aims to automatically verify software stacks and
aggregate relevant observations into Thoth’s knowledge database.

## Dependency Monkey use cases

Dependency Monkey was designed to automate evaluation of certain aspects of a
software stack, such as code quality or performance. Two main aspects will be
derived from software stacks as observations. Observations are currently
consumed by Thoth for it’s recommendations/advises: Does the given software
stack build on a given Operating System ( on the targeted hardware (if any
hardware requirements such as specific CPU are provided for builds)?  What are
runtime characteristics of the given software stack on targeted hardware (if
not requested a specific one, any hardware available is used to test the
software stack) An example of runtime characteristics could be performance
indicators or indicators such as accuracy in case of math libraries.

## Dependency Monkey architecture design and overview

Dependency Monkey is made out of multiple parts used in the project Thoth:

* Management API (optional)
* Dependency Monkey job
* Amun API
* Inspection builds and Inspection jobs
 
## Example Use of Dependency Monkey

To illustrate dependency monkey use, let’s create a fictional use case. A first
interaction point can be /dependency-monkey/python endpoint exposed on Thoth’s
Management API service (there is currently supported only Python ecosystem).
This endpoint accepts input from a user which is in a form of a software stack
that should be validated. The software stack input is compatible with Pipfile
as produced by [Pipenv](https://pipenv.readthedocs.io) that can assign
different package indexes to packages so dependency monkey can verify Python
packages across different Python package indexes.

Besides software stack, the dependency-monkey endpoint accepts a base image
that should be used for software stack validation and verification. Currently
supported distributions span across those using  dnf, yum and apt for native
packages. There can be optionally passed a list of native packages that should
be installed into the given base image on build time (for example pipenv for
testing pipenv provided in Fedora as rpm). Optionally a URL or a script can be
passed  and will be used to validate the given software stack. See
OpenAPI/Swagger specification of [Management
API](https://github.com/thoth-station/management-api) to see all the possible
options.

Once the dependency-monkey endpoint is triggered with the requested input, a
“Dependency Monkey” job is created  in one of the Thoth’s namespaces reserved
for Dependency Monkey runs (pre-allocated pool of resources).

The Dependency Monkey job creates in-memory structure that can generate all the
possible software stacks given the software stack specification in Pipfile. The
resolution algorithm utilizes core pip resolution mechanism to guarantee the
result to be fully compatible with Python’s pip. The actual dependency
management is however not done by pip or Pipenv. An off-line dependency
resolution algorithm was designed in support with pre-computed data available
in the graph database. This makes resolution much faster in comparison to
native pip or Pipenv resolving (note there can be thousands or even millions of
possible software stack combinations).

The in-memory structure is capable of generating fully pinned down software
stacks as produced by pip or Pipenv in real world. These software stacks are
subsequently sent to Amun API service for inspection (along with context given
on the dependency-monkey management API endpoint such as base container image,
native packages, ...).

Amun is a service that is capable of building software stacks considering the
base container image and native packages that are requested to be installed.
This procedure is done in the build inspection jobs triggered by Amun API in
inspection namespace with resources dedicated for inspection runs. OpenShift
builds and image streams were reused for building resulting container images.

If a user supplies a test script, an inspection job is created. The job goal
is to verify the given software stack. The very first step done by the
inspection job is gathering hardware information about runtime hardware present
(currently only CPU related information; this is done in an init container).
The inspection job after that executes user provided script (e.g. a Python
script) in the given environment and gathers all the relevant information in a
form of a JSON object (e.g. standard error, standard output, script’s status
code) - these information are called inspection job report.

These reports are gathered by a job triggered by an operator and placed onto
Ceph as JSON documents. Besides that, results are gathered, parsed and
observations are feeded into the graph database where these observations are
used by Thoth’s recommendation engine to recommend high quality software stacks
based on gathered observations.  If a user wants to perform an inspection on a
pinned down software stack without running the Dependency Monkey job to
generate software stacks, the thoth-adviser  CLI can be utilized.
[Thoth-adviser](https://github.com/thoth-station/adviser) provides submit-amun
subcommand to trigger Amun API to run the inspection job and build (or
OpenAPI/Swagger UI can be used directly).

![Architecture](https://raw.githubusercontent.com/thoth-station/adviser/master/docs/dm.png)