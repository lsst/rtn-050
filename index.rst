:tocdepth: 1

.. sectnum::

.. Metadata such as the title, authors, and description are set in metadata.yaml


Abstract
========

This technote describes the means by which the desired DP0.3 dataset will be served through the services of the Rubin Science Platform.
Initially it will serve as a implementation design baseline and then evolve into an as-built description.

Motivation and Overview
=======================

Ticket `PREOPS-1152`_ and a recent revision of `RTN-011`_ envision the creation of a new entry in the "DP0"
(simulated data) series of Data Previews:
service of an existing catalog-based "fast simulation" of the expected Solar System data from the LSST survey,
not tied to image data nor to any output of the LSST Science Pipelines.

This "DP0.3" will provide some exposure to the Rubin Science Platform (RSP) and the planned Solar System data model to Data Preview delegates, and allow the early testing of certain features of the RSP that are specific to Solar System (or, more precisely, moving-object) data.
Data Preview 0.3 is planned for release to users sometime between June and September 2023 (see `RTN-011`_).

The fast simulation, generated by the Solar System Science Collaboration (SSSC), represents the results of the full ten-year baseline LSST survey.
Four catalog tables involved with Solar System science and the Rubin moving-object pipeline will be served as part of DP0.3:
``DIASource``, ``SSSource``, ``SSObject``, and ``MPCORB``.
The first two tables will each have approximately one billion rows each, and the last two will have approximately ten million rows each.

The proposed exercise has some notable limitations compared to the final expected environment for Solar System data:

- As the data are the result of a "fast simulation" without imaging, none of the capabilities of the RSP dealing with imaging data will be exercised.
  For the most part this is not a concern, since those capabilities are currently being exercised by DP0.2 and will be further explored by the future DP1 and DP2, but there is at least one relevant special case:

  - In the operations era, the RSP will supplement the ability to retrieve time-series data for moving objects
    with an associated ability to retrieve corresponding cutouts from the images on which the points in the time series were observed,
    i.e., following the object across the sky.
    Because of the lack of imaging data, this will not be possible in DP0.3.
  - More generally, the usual linkages from single-epoch measurements in catalog tables (in this case, ``DIASource`` and ``SSSource``)
    to the associated images will not be available.

- As the DP0.3 dataset is the result of a static simulation, rather than a progressive nightly release through the PPDB, as we will have in Operations,
  DP0.3 will not exercise certain dynamic aspects of the data model, such as the transformation from DIAObject to SSObject when
  sufficient data are accumulated to allow identification of a previous DIAObject as arising from an SSO.
  It will therefore also not exercise the ability of users of the RSP to follow and/or replay that evolution.

- In the real system, it will be possible to provide links from the Rubin-hosted tables directly to corresponding
  external queries against the MPC.
  Since the DP0.3 data include simulated SSOs, this will not be possible to exercise against the real MPC.

.. _PREOPS-1152: https://jira.lsstcorp.org/browse/PREOPS-1152
.. _RTN-011: https://rtn-011.lsst.io/

Input Data
==========

The simulation itself is described at https://github.com/lsst-sssc/lsst-simulation .
It produces the following tables, all of which have reference definitions in the `DPDD`_:

.. _table-ssotables:

.. table:: Major tables to be included in a Solar System DP0.3.  

   +-----------+--------------------------------------------------------+--------------+-------------------+---------------------+
   | Table     | Purpose                                                | Size in rows | Number of columns | Data volume on disk |
   |           |                                                        |              | (actual/DPDD)     | (in XXX format)     |
   +===========+========================================================+==============+===================+=====================+
   | DIASource | Transient observation in a single difference image     | ~1B          |                   |                     |
   +-----------+--------------------------------------------------------+--------------+-------------------+---------------------+
   | SSSource  | Apparition of a reconstructed SSO in a single image    | ~1B          |                   |                     |
   +-----------+--------------------------------------------------------+--------------+-------------------+---------------------+
   | SSObject  | Reconstructed SSO with orbital parameters              | ~10M         |                   |                     |
   +-----------+--------------------------------------------------------+--------------+-------------------+---------------------+
   | MPCORB    | Shadow of Minor Planet Center database                 | ~10M         |                   |                     |
   +-----------+--------------------------------------------------------+--------------+-------------------+---------------------+
   | (total)   |                                                        |              |                   | 0.5 - 1.0 TB        |
   +-----------+--------------------------------------------------------+--------------+-------------------+---------------------+

The simulation is based on a specific, chosen OpSim run, which will be documented.
The actual simulation output to be used for DP0.3 is yet to be produced, but a prototype dataset exists and has been in use by the SSSC, ingested into a PostgreSQL database.

Stretch goal: Visit-outline table
---------------------------------

As noted above, the existing SSSC data for DP0.3 is based on a catalog simulation,
with no images involved in the data chain.
However, the simulation _is_ based on a survey simulation and therefore on pointings and
sky coverage from that simulation.

It might improve the user experience, as well as allow the exercise of additional RSP capabilities,
to include a simple visit table, in ObsCore format, including the pointings and sky coverage of the
survey on which the catalogs are based.
This would allow, e.g., visualizing DIASource and SSSource table contents against a background of the
visit image outlines on the sky, using existing RSP capabilities.

This table would serve as a partial prototype for the ObsLocTAP service required in operations,
as ObsLocTAP is based on an elaboration of the ObsCore data model.

Delivering this feature is not a requirement for DP0.3.
It can be added after the initial release, resources permitting.

.. _DPDD: https://lse-163.lsst.io/

Service Model
=============

The RSP instance for DP0.3 will be in the  Interim Data Facility (IDF) hosted on the Google cloud platform, and, we argue below,
could and likely should share most of its components with the existing data.lsst.cloud RSP instance for DP0.2.

The catalogs will be held in a PostgreSQL database, most likely on-premises at the USDF,
in a preview of the "Hybrid Model" for the US DAC.

Back-end data for DP0.3 as envisioned herein is limited to catalogs.
Therefore no image file store is required, no image metadata service (e.g., ObsTAP), and no DataLink "links service".

The specific components that *are* required follow:

Catalog Database
----------------

Qserv is not a suitable choice for the database back end, for two reasons:

- In the operations-era system, the SSO tables will live in the "Prompt Products Database" (PPDB), which is baselined as PostgreSQL.
- A partitioning strategy for Qserv is unclear, since SSObjects are not spatially localized, so supporting efficient queries on the
  SSObject-to-SSSource relationship might be in conflict with efficient spatial queries on the SSSource table.

Based both on SSSC experience and expert opinion, a single PostgreSQL server should be adequate for the datasets envisioned.
To support spatial searches via (CADC) TAP, the ``pgsphere`` extension should be installed on the Postgres server.

In the "Hybrid Model" for the US DAC, the user-facing services will be in the Google cloud, with the data back ends at the USDF.
Implementing this model for DP0.3 will require a Postgres server at SLAC,
and ensuring that a cloud-based TAP service can reach that server for queries.
This is the likely baseline for DP0.3; we will analyze the feasibility of this in the near future.

An alternative would be to configure a Postgres service at the IDF (Google cloud).
Some research will be required to determine whether a sufficiently large Postgres service can be configured easily in the Google cloud.
Pre-configured versions of Postgres with ``pgsphere`` installed are not currently available from Google.

If (see above) an additional ObsLocTAP-style table of visits proves desirable, this can be included in the database.

Data Services
-------------

TAP service
^^^^^^^^^^^

If the database is in Postgres, the CADC TAP service should be used.
CADC's code base has native support for Postgres back ends.
The work done in December/January 2022/23 to deploy a Postgres-based TAP service for the "live ObsTAP" instances should be applicable.
The same DataLink-support extensions to CADC TAP that were developed by SQuaRE for the Qserv-backed TAP
implementation will be needed for DP0.3 as well.

At present we do not have the ability to support multiple back ends from a single TAP service instance,
so DP0.3 will require its own TAP endpoint even if it is otherwise incorporated into data.lsst.cloud
alongside DP0.2.
For instance, "data.lsst.cloud/api/ssotap" might be a suitable name.
(Future work on the RSP is expected to address this limitation, but not on the DP0.3 timescale.)

"TAP_SCHEMA" data for the service will be obtained from Felis in the usual way,
most likely with a DP0.3-specific Felis file in the ``sdm_schemas`` repository.
The RSP Scientist, Gregory Dubois-Felsmann, will develop this file, based on existing DPDD Felis code,
to reflect the precise DP0.3 data model, in collaboration with the SSSC experts on the dataset.

This work includes providing descriptions, units, UCDs, and foreign-key annotations showing the links
between tables in the data model.

DataLink services
^^^^^^^^^^^^^^^^^

As noted above, no DataLink "links service" for images is required or even relevant to DP0.3,
and no other DataLink services are required in order to meet the core goals for this Data Preview.

However, "one-line" query-rewriting services designed for use with DataLink will be desirable
to enable convenient user access to actions like "show me all the SSSources for this SSObject".
Such services rewrite a simple REST API query for, e.g., an SSObject ID to a TAP query with
the appropriate corresponding ADQL text.

The existing ``datalinker`` framework will be suitable for these services, and experience
with that framework has shown that a new "rewrite" service of this nature can be configured
very quickly.
For this reason we are confident that appropriate services of this nature can be included
in the DP0.3 baseline.

The specific set of services to be provided will be discussed with the SSSC during the
development phase of DP0.3.

Stretch goal: Predicted-position service
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

The same Rubin-developed service framework for deploying IVOA-style data services could
readily be used to create a service that returned predicted positions for Solar System objects,
based on their orbital parameters, in all visits in which they would be within the field of view.

The SSSC has code that can perform the underlying computation.
Deployment of a service in the RSP framework based on that code would serve as a prototype
for future services satisfying the RSP requirement DMS-PRTL-REQ-0099, "Overlay LSST-Derived Orbits".

We will investigate providing this as an upgrade to DP0.3 after its initial release.

.. Note to the reviewer - this may not turn out to be relevant to DMS-REQ-0323, which may
   be read as being about actual observations, not predictions.  However, there's no reason
   we couldn't include parameters like the phase angle in the table of predictions.

User Interface Services
-----------------------

Portal Aspect considerations
^^^^^^^^^^^^^^^^^^^^^^^^^^^^

We will need to decide whether to include DP0.3 in the same RSP instance as DP0.2.
At this time, we are assuming that will be the plan.

In that model, because of the limitation on multi-back-end TAP services, users will have to be
given a choice between DP0.2 and DP0.3 at the top of the TAP query screen in the Portal Aspect.
This is an existing capability of the Portal (see :ref:`fig-portal-tap-menu`).
Note that this requires one or the other to be the default, so, unless additional work is
requested, it might turn out to be the case that DP0.3 users have to always start their
session by changing TAP services.

.. figure:: /_static/Portal-TAP-menu.png
    :name: fig-portal-tap-menu
    :target: ../_images/Portal-TAP-menu.png

    Existing TAP service selection menu in the RSP Portal Aspect.

Once the TAP service is selected, the user will be presented with a menu of available tables.
The presentation order of tables, and of columns within tables, are controlled by the Felis-based
TAP_SCHEMA metadata mentioned above.
This permits the optimal order of tables and columns to be set in consultation with the SSSC.

The Portal Aspect displays all spatially-organized tabular query results against a default
context image, generally a HiPS map.
In DP0.2, we have changed that default context image to be a HiPS image of just the DC2 field.
This was important as DP0.2 exists in a simulated universe not based on the real sky.

For DP0.3, a real sky is appropriate as the context image.
If DP0.3 is in the same RSP instance as DP0.2, we will have to develop a means of associating
the default context image with the selected TAP service, to avoid users having to manually
change context images in every session.
We would likely use a color HiPS image from 2MASS as the default context image for DP0.3,
unless the team has a preference for a different existing all-sky HiPS (e.g., from PanSTARRS).

If this development is not possible in time for the timely initial release of DP0.3,
we can add it as a subsequent upgrade.


Notebook Aspect considerations
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

We anticipate that most users of DP0.3 will focus their work in the Python-based Notebook Aspect
environment.

We note that users will have to migrate from whatever they may be doing in the existing SSSC
environment (perhaps SQLAlchemy?) to the use of TAP queries.
This has been extensively explored in DP0.2 (albeit over Qserv), so we don't anticipate any
issues, with the following one exception:

The existing Python "convenience function" for obtaining a reference to the RSP TAP service
from within a Notebook Aspect notebook, with the necessary authentication information
embedded automatically, does not currently support there being more than one TAP service
per RSP instance.
Therefore, if DP0.3 is released in the same instance as DP0.2, which will require the use of
two TAP services in the same instance, as noted above, some work will be required to
generalize this.

Authentication and Authorization
--------------------------------

At the moment we are not aware of any special restrictions on access to the SSSC simulation,
so the baseline would be to make all DP0.2 and DP0.3 data accessible to the same set of users
and base it on the CILogon/COmanage IAM mechanism to which DP0.2 is in the process of being transitioned.


Timeline
========

- January 2023: Refine definition of this Data Preview.
  Review existing SSSC data model for any changes needed for DPDD and/or standards (e.g., IVOA)
  conformance (Dubois-Felsmann, Juric).

- February 2023: Regenerate simulation (Juric).
  Produce initial Felis data model (Dubois-Felsmann).
  Prepare necessary USDF infrastructure (R. Dubois).

- March 2023: Initial version of data turned over to USDF team.
  Ingest into Postgres at USDF (Mueller).
  Fix/regenerate source data if problems are found.

- April 2023: Establish TAP service in Google Cloud over USDF Postgres DB (SQuaRE).
  Complete Portal (context image selection) and Notebook (TAP service helper function) software refinements.

- May 2023: Exercise DP0.3 internally.
  Complete DataLink microservices and metadata deployment (SQuaRE and D-F).
  Develop tutorials and notebooks (Community Engagement Team (CET)).

- 26 May 2023: Release candidate turned over to product owners (Dubois-Felsmann and Slater).

- 13 June 2023: Nominal release date.

- June-September 2023: Public (RTN-011) commitment for release date.


Preparations Required
=====================

Regeneration of the Simulation
------------------------------

An initial review of the existing SSSC simulation, performed last year by Gregory Dubois-Felsmann
and Mario Juric, exposed some minor issues in consistency with the DPDD and other aspects of the
schema.
Mario described the changes that were suggested as very easy to make.

The SSSC have stated that they want to regenerate the simulation based on the recently released
V3.0 of the baseline survey (see `PSTN-055 <https://pstn-055.lsst.io/>`__).

The first step in preparation for DP0.3 will therefore be to confirm the changes needed and then
proceed to re-run the simulation.
If a visit table is decided to be a useful adjunct to DP0.3, this would be the time to define
and generate it.

Database Setup
--------------

The USDF team at SLAC and Fermilab will establish the necessary Postgres database,
taking into account the need to access it from the TAP service in the Google-cloud-based IDF.

Ingest
------

On `PREOPS-1152`_, Mario Juric reports that:

"For our internal use, we've used `pg_bulkload <https://ossc-db.github.io/pg_bulkload/pg_bulkload.html>`__
to rapidly (in ~30 minutes) ingest these tables into a database.
The details are in `this (messy) notebook. <https://github.com/mjuric/ssp-ddpp/blob/master/daily-data-products-pipeline.ipynb>`__
Using more typical loading mechanisms (from .csv files, etc.) is not an issue, just will be slower.

"If a postgres database can be set up within the RSP,
with pg_bulkload enabled and given administrative permissions I would be able to load these data into it probably in a ~few days.
This setup would also allow for uploads of future dataset updates:
we refresh these simulations ~annually, as new baseline simulations become available and the software is improved."

However, our initial baseline for DP0.3 is to use more conventional loading mechanisms.

Data Model Metadata
-------------------

Felis code for the planned tables will be generated, initially during February 2023,
by Dubois-Felsmann, based on a combination of existing DPDD-based Felis code and the
actual schema of the SSSC simulation.

Service Deployment
------------------

The TAP service and any associated DataLink services will be deployed using the usual
``phalanx`` mechanism for configuration of the RSP.
The only new development for DP0.3 will be the configuration of the TAP service to connect
to the Postgres database back end at the USDF, with any associated security-related work
that may require.

SLAC IT security staff will be consulted early in the process to ensure that any
concerns are addressed before the deployment is expected to go live.

.. See the `reStructuredText Style Guide <https://developer.lsst.io/restructuredtext/style.html>`__ to learn how to create sections, links, images, tables, equations, and more.

.. Make in-text citations with: :cite:`bibkey`.
.. Uncomment to use citations
.. .. rubric:: References
..
.. .. bibliography:: local.bib lsstbib/books.bib lsstbib/lsst.bib lsstbib/lsst-dm.bib lsstbib/refs.bib lsstbib/refs_ads.bib
..    :style: lsst_aa
