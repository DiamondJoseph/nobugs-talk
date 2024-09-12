# Not-invented-here: Building an acquisition platform with off-the-shelf components

## Introduction

<img src="./pictures/gda_logo.jpg" alt="The GDA logo: G, D and A in a sans-serif font with a grid of squares viewed from an oblique angle." height=100/><img src="./pictures/athena_logo.png" alt="The Athena platform logo: a 'boggle faced' owl staring into ones soul." height=100/>

Diamond Light Source has deprecated the GDA (Generic Data Acquisition) platform and is replacing it with Athena, a service-based acquisition platform centred on the bluesky data collection framework from NSLS-II. Historically Diamond has struggled to overcome a mindset that every aspect of the operation of a Synchrotron is equally and inextricably unique. This poster seeks to dispel that myth and describes experiments that have occured and are ongoing which make use of pre-existing free open-source software, with effort spent on altering Diamond requirements or contributing to the existing: rather than "reinventing the wheel".

GDA is a previously open-source Java monolithic server/client appplication. The source was closed after changes in the build system required the Diamond filesystem was available to build and run the product and the available source was frozen at this point. Visiting users were able to contribute code written in Jython (Python implemented in the Java Virtual Machine) and only as specific alterations to existing experimental proceedures.

Athena is a service-based acquisition platform which is aiming to resolve several identified shortcomings of the GDA platform: lack of authentication & remote access; allowing user modification of scan proceedures; Jython being limited to Python 2.7; improving maintainability of a mature and expansive codebase.

## Time Series Experiment

In November 2023, an initial experiment was performed with a minimal Athena stack:

* GDA's Jython console was used as a client for blueapi, which ran in a beamline-local Kubernetes cluster.
* A PandA device, Pilatus detector, TetrAMM ammeter, Aravis camera and Linkam temperature controller were created in ophyd-async.
* A bluesky plan was written for the experimental proceedure which created and write a Sequence Table to the PandA; configured the Linkam to perform a stepped scan down in temperature, then target a high temperature; enabling the PandA to periodically triggered the other devices to collect while the Linkam went to the high temperature.
* The bluesky RunEngine controlled the experiment and emitted bluesky event-model documents which were consumed by a NeXus service written to convert from the event-model scheme to Diamond's expected NeXus structure.

### ophyd-async

Diamond's I22 beamline previously utilised a Time Frame Generator for time series experiments, however the device was end of life and at risk of failing. GDA had good support for hardware triggered experiments using the PyMalcolm<sup>[1]</sup> middleware layer but did not neatly support variable period scanning.
Diamond brought expertise in the field of hardware-triggered scanning to the bluesky collaboration in the form of ophyd-async<sup>[2]</sup>, which revisited some assumptions made in the original ophyd library<sup>[3]</sup>, the oldest of the libraries in the organisation. Ophyd amounts to ~22,000 non-test lines of code: ophyd-async currently consists of ~7,000, while meeting most existing use cases, and unlocked the use of the core bluesky library<sup>[4]</sup> (~19,000 LOC).

<sup>[1]</sup><https://github.com/DiamondLightSource/pymalcolm>
<sup>[2]</sup><https://github.com/bluesky/ophyd-async/>
<sup>[3]</sup><https://github.com/bluesky/ophyd>
<sup>[4]</sup><https://github.com/bluesky/bluesky>

### blueapi

GDA historically operated as a server/client model, with the client running either as a user or a beamline account while the server ran as a privileged user able to write to the shared filesystem. This contrasts with the usual mode at NSLS-II of running bluesky directly from the command line, with data presented by a data API. To allow a continuation of service  blueapi: which combines a bluesky RunEngine; a REST API; and necessary context mapping between them was produced.

Blueapi's REST API is served by the FastAPI library for python: together with the Pydantic data validation library to construct JSON schemas for experimental proceedures, this enables the service to be wholly written in modern Python. Blueapi is currently <3000 non-test lines of code: excluding test code, the Yaml templates for its helm deployment are on near-parity, but provides a crucial connection between the served API and running experimental proceedures.

<img src="./pictures/blueapi.svg" alt="A stack of boxes, with FastAPI (an open source library) at the top, below which is blueapi (an open source library which has been 'invented here' at Diamond) and below which is the RunEngine from the bluesky open source collaboration. The RunEngine connects to a series of devices: a TetrAMM ammeter which was written specifically for Diamond's needs, but also a PandA device and Pilatus detector, which are generic devices contributed to the ophyd-async collaboration."/>
*Blueapi is the glue between FastAPI and the RunEngine. Some device classes were written for the experiment, but others were contributed generic devices.*

### NeXus & Analysis

### Deployment & Maintenance

The Athena stack was deployed with Helm on a Kubernetes cluster local to the beamline by utilising a Helm Umbrella chart with components as dependencies: blueapi, the NeXus service and a RabbitMQ instance<sup>[1]</sup>. This enabled us to deploy the stack as a coherent block of functionality.

Moving GDA messaging into Kubernetes was an existing desire to address observed reliability problems by making use of monitoring and service restarting, however no up-to-date supported ActiveMQ Helm deployment could be found. As GDA internal messaging was utilising JMS, RabbitMQ was deployed with a JMS plugin and changes to the GDA core libraries to enable switching between ActiveMQ and RabbitMQ were made.

<img src="./pictures/kubernetes_logo.png" alt="The Kubernetes logo: a 7 pointed ship's wheel." height=100/><img src="./pictures/helm_logo.svg" alt="The Helm logo: a stylised 8 pointed ship's wheel occluded by the work HELM." height=100/>

<sup>[1]</sup><https://artifacthub.io/packages/helm/bitnami/rabbitmq>

## Stopflow Experiment & towards Athena 1.0

A 2nd experiment, initially due to occur June 2024 but delayed due to hardware problems to March 2025 will again make use again of time-based PandA hardware scanning.

* FastAPI autogenerated API page used as client to run the experiment from a web browser, enabling browser-based OIDC flow.
* Previous devices updated to ophyd-async 0.5 patterns, devices created for background beamline readings (mirror positions, synchrotron current etc.)
* A bluesky plan was written for the experimental proceedure:
  * Capturing background devices at start and end of scan
  * Creating and writing a Sequence Table to the PandA to capture N frames, then await a hardware trigger and collect M additional frames
* A NeXus device metadata service created and utilised by the existing NeXus service to get invariant data about devices involved in the scan (e.g. pixel size of the Pilatus detectors).

### Tracing and Metrics

GDA, as a monolithic application, was simple to track operations and debug locally. Transitioning to a service based architecture loses simplicity of debugging. Tracing calls through the system, in addition to logging within the services, enables introspection of the whole system for debugging and enables collection of metrics. Tracing is being implemented with Open Telemetry<sup>[1]</sup>, Jaeger<sup>[2]</sup> and Prometheus<sup>[3]</sup>

Below an example experiment is traced through blueapi, RabbitMQ and the legacy NeXus service, plotted with Jaeger: blueapi takes ~1ms to publish new data, and is expected to do so at ~10Hz. In this example, the NeXus service takes just over 100ms to consume per-point data and write it to disk, suggesting that it may eventually become a bottle neck.

This example gives some easy places to look for efficiency gains: deserialising the StartDocument took much longer than similar operations, it may benefit from an additional pass. If in a larger dataset it was a standout, or if the length of the deserialising obviously changed after a new version was released, it may also warrant investigation. Gathering the data allows us to make reasoned decisions.

<img src="./pictures/tracing.jpg" alt="A trace of a simple experiment run with blueapi and consumed by the NeXus service: blueapi emits 4 documents: a Start document with scan metadata; a Descriptor document which describes devices involved in the scan; an Event document with point data from the described devices; a Stop document with metadata about how the scan finished. These occur over ~70microseconds. After about a 100microsecond gap, during which the events are sent to a message bus and consumed, the NeXus service consumes the documents and write them to a NeXus file on disc. Some of the larger spans are commented: deserialising the StartDocument takes ~50microseconds, while creating the NeXus file on disc at the first point takes ~100microseconds."/>
*An example OTEL trace viewed in Jaeger*

<sup>[1]</sup><https://opentelemetry.io/>
<sup>[2]</sup><https://www.jaegertracing.io/>
<sup>[3]</sup><https://prometheus.io/>

### ArgoCD

<img src="./pictures/argo_deployment.png" alt="ArgoCD management console for the p45 beamline at Diamond: the beamline is shown as synced and healthy: the app-of-apps is shown at the left, branching into each of the services deployed for it: 4 IOCs for detectors, motion controllers; the NeXus service, blueapi and a RabbitMQ instance; and common infrastructure for the IOCS."/>

The Helm umbrella chart makes assumptions about interoperability requirements of the services, and makes atomic changes and maintenance more awkward than intended. Initial moves in the direction of deploying services with ArgoCD are underway, with aims of having beamline state represented by 2 repositories: one containing the state of what is deployed<sup>[1]</sup>, and another representing the state of the deployed software<sup>[2]</sup>.

ArgoCD<sup>[3]</sup> is a Kubernetes operator which operates similarly to Helm but allows for a combination of deploying multiple services (similarly to the prior Helm Umbrella chart) while also rolling updates individually to those services, by deploying as an app-of-apps:<sup>[4]</sup> in the linked examples, p45-deployment<sup>[1]</sup> defines the app while p45-services<sup>[2]</sup> defines the apps.

<sup>[1]</sup>example: <https://github.com/epics-containers/p45-deployment>
<sup>[2]</sup>example: <https://github.com/epics-containers/p45-services>
<sup>[3]</sup><https://argo-cd.readthedocs.io/en/stable/>
<sup>[4]</sup><https://argo-cd.readthedocs.io/en/stable/operator-manual/cluster-bootstrapping/#app-of-apps-pattern>

### AuthN/AuthZ

Authentication (knowing which user made a request) and Authorization (deciding whether the user is allowed to make a request) becomes even more important as data collection is no longer limited to on-site.

Athena has been developed with auth concerns from its conception, however infrastructure has been delayed. AuthN is anticipated to be in place by the end of the year, with the user logging in to blueapi through an OIDC flow<sup>[1]</sup>, either as a command line tool or through a web front end, with authorization provided by Open Policy Agent (OPA)<sup>[2]</sup> or a similar authorization provider.

A pair of example flows which are being considered are shown:

The first makes use of FastAPI middlewares<sup>[3]</sup> to authenticate the user against an external OIDC provider (existing authentication infrastructure) and authorize the call for the authenticated user. In this arrangement, the size and complexity of blueapi is slightly increased and all requests are processed inside the service.

<img src="./pictures/middleware.png" alt="Authentication and Authorization enabled by use of middleware inside the blueapi instance. Traffic reaches blueapi, is inspected for authentication headers and the user is redirected to the OIDC provider if not present, else the headers are checked for validity with the provider. Once authenticated, the request is passed by the authentication middleware to the authorization middleware, which makes a decision based off of authorization information. The middlewares and OIDC providers are standard tools, but the authorization information requires domain specific knowledge."/>
*Auth/Auth enabled by use of middleware bundled into blueapi*

The second makes use of Sidecars<sup>[4]</sup>: a service mesh (e.g. Istio<sup>[5]</sup>) deploys all pods with additional containers that can intercept incoming traffic and pre-process it, including redirecting undirected traffic to the OIDC provider, or authorizing the call with an external authorization provider, such as an instance of OPA.

<img src="./pictures/sidecars.png" alt="Authentication and Authorization enabled by use of sidecars alongside the blueapi instance. Traffic is intercepted prior to blueapi, but the flow is otherwise the same as before. The domain specific knowledge for authorization is shown as an example OPA Bundle Server."/>
*Auth/Auth enabled by use of a ServiceMesh in the Kubernetes cluster*

The decision of which architecture will be adopted is a pending question: if multiple services require to authenticate, the service mesh is repeatable and language agnostic; however it adds initial complexity and configuration, and is confined not just to deployment with Kubernetes, but mandates particular configuration.

<sup>[1]</sup><https://openid.net/developers/how-connect-works/>
<sup>[2]</sup><https://www.openpolicyagent.org/>
<sup>[3]</sup><https://fastapi.tiangolo.com/tutorial/middleware/>
<sup>[4]</sup><https://kubernetes.io/docs/concepts/workloads/pods/sidecar-containers/>
<sup>[5]</sup><https://istio.io/latest/about/service-mesh/>

## Next steps

### Client Generation

We are currently exploring how to offer generated GUIs for plans exposed by blueapi to contribute resources to the bluesky-webclient library<sup>[1]</sup>. The below example extracts the docstring from a python plan<sup>[2]</sup> and provides appropriate fields for the type using JSONForms<sup>[3]</sup> and Pydantic<sup>[4]</sup> data validation.
While beamlines that operate a standard experimental proceedure may wish to specialise their UI components, blueapi will provide at minimum a viable generic UI and is increasingly aiming to enable rich UI generation<sup>[5][6][7]</sup> by making use of Python annotations and Pydantic Field objects<sup>[8]</sup> which are all optional inclusions.

<img src="./pictures/client.jpg" alt="Autogenerated client for a blueapi instance. On the left, a dropdown box presents available plans: scan, set_absolute, set_relative and others. On the right, a header 'Basic linkam_plan plan UI' is above a large docstring explaining the parameters of the plan. Below, a series of form boxes with numbers allow filling the parameters of the plan function."/>
*Autogenerated UI using JSONForms for the plan run in November 2023*

<sup>[1]</sup><https://github.com/bluesky/bluesky-webclient>
<sup>[2]</sup><https://github.com/DiamondLightSource/i22-bluesky/blob/1f0a00828b006388d1a56b0f426610ab8d7bfcdf/src/i22_bluesky/plans/linkam.py#L50>
<sup>[3]</sup><https://jsonforms.io/>
<sup>[4]</sup><https://docs.pydantic.dev/>
<sup>[5]</sup><https://github.com/DiamondLightSource/blueapi/issues/518>
<sup>[6]</sup><https://github.com/DiamondLightSource/blueapi/issues/519>
<sup>[7]</sup><https://github.com/DiamondLightSource/blueapi/issues/616>
<sup>[8]</sup>example: <https://github.com/DiamondLightSource/i22-bluesky/pull/90/files>

### Widening Adoption

Ophyd-async will soon support N-dimensional trajectory scans through use of the ScanSpec library<sup>[1]</sup>: once this is in place, it is hoped that many of the experimental proceedures currently in use at Diamond can be replicated. A widening library of ophyd-async device and behaviour support in both the library itself and in dodal<sup>[2]</sup>, the Diamond device library, alongside beamline-specific code<sup>[3]</sup> and technique-specific code<sup>[4]</sup> is expected to be responsible for new projects at Diamond and an increasing proportion of business-as-usual.

<sup>[1]</sup><https://github.com/bluesky/scanspec>
<sup>[2]</sup><https://github.com/DiamondLightSource/dodal>
<sup>[3]</sup>example: <https://github.com/DiamondLightSource/i22-bluesky>
<sup>[4]</sup>example: <https://github.com/DiamondLightSource/mx-bluesky/>

### Data API & h5py-based NeXus

Existing Diamond data analysis pipelines rely on NeXus SWMR (single write, multiple read) to perform analysis of data as it is written to the NeXus file. For Diamond-II flagship beamlines coming online in the next few years analysis will instead be either from a data API (likely bluesky/tiled<sup>[1]</sup>) or by consuming the bluesky event-model documents directly. This decouples the NeXus file from needing to be written "live" during the experiment, and provides an opportunity to rethinking priorities of writing the file: if the file is written as a post-processing step, then results of processing can be included to make consuming the data simpler, and archiving the file more efficient.

It is intended that the legacy NeXus service will be retired after event-model document storage is finalised.

<sup>[1]</sup><https://blueskyproject.io/tiled/index.html>
