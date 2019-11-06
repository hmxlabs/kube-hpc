# Multi Cloud Compute Farm
## Introduction
A number of our clients run large scale grids for HPC workloads, to date we have yet to see any substantial differences in the architectures employed or any additional business value through the use of a proprietary system for the distribution of workloads across a large number of cores. (The analytics that runs of the cores is of course another matter and not the subject of this document).

The various architectures have also been largely similar with the differentiation largely to due to the integration patterns required by other systems in the enterprise.
After having repeated this same architectural principal multiple times, I grow tired of drawing the same diagrams to explain what is missing, broken or subject to optimisation in the current system and believe it is time to just open source the general architecture.

The architecture described here is very deliberately technology agnostic and could equally well be implemented using proprietary grid platforms such as TIBCO GridServer or IBM Spectrum Symphony or using open source only and building out the distribution technology on top of Kubernetes. (Please see our kube-hpc project for more on that).

There is clearly a fair amount of implementation required to deliver this architecture, especially to meet the hybrid multi cloud requirement. While we hope eventually to build it as an open source effort unless we find a client that is willing to pay for this it likely to be an outdated model before we complete the build out. 

As such this is largely here as a reference (and proven) architecture which has been scrutinised by a number of people with decades of experience of designing, writing, maintaining and running very large-scale compute grids.

## Gaps
There are a number of gaps to hybrid multi cloud setups architectures that we don’t yet have the tooling to address. These are notably around the operational requirements such as logging and monitoring.

The log aggregation and monitoring tools currently available operate on the principle that network capacity to perform the data aggregation (be that logs or monitoring output) is freely available. (as the assumption if the entire system operates within one cloud provider or data centre). Under a multi cloud environment this is in the best case an expensive proposition and in the worst case not at all feasible. 

Yes, even in an era with Inifiband and dedicated links to cloud providers, when these links need to be shared across an entire enterprise and also handle the workload for over 100,000 cores trying to send the log output (from the aforementioned 100,000 cores) back to on premises (or a central cloud instance) is quickly a futile effort.

The sensible option is of course to leave log output within the host cloud and send only aggregate monitoring data. The tooling to then view logs as though they were centralised and pull on demand though is not yet available. Similarly, the ability to have monitoring output at a detailed level available to view in a single place but pull on demand is missing.

## Architectural Principles
The entire architecture is designed, ground up, to be run containerised via a container management framework such as Kubernetes (or other distributions of the same such as RedHat OpenShift). This normally implicitly entails certain architectural paradigms, however for the sake of clarity some of the implications of this are detailed here.

Additionally, the system must be capable of running in a hybrid multi cloud environment. This may mean ordinarily operating with base load on premises with peak and spike loads running the cloud but could also be implemented as the entire stack running on (multiple) public cloud(s). It goes without saying this needs to be achieved without vendor lock in to any specific cloud provider or product.

## Compositional Architecture
The system is comprised of a large number of small components. Functionality is created through the composition of many small components with each component existing as a fully independent unit. It would not be unreasonable to see components that consist of only several hundred lines of code, i.e. very limited functionality. 

Each component exists in its own right as an individually source controlled, buildable and releasable unit with its own continuous delivery pipeline.

Think GNU utilities on Linux. Composition of several small very specific tool (e.g ps, grep, kill as a common combination) to achieve the desired outcome. This approach differs starkly to traditional development practices which are very much geared around building large scale services (and are still very much in use in many of the organisations that require large scale HPC).

## Stateless
It is not uncommon for components in a system to be designed to be stateless to allow for easier scaling and as such all components within the architecture aspire to be stateless wherever possible. State will be maintained in dedicated caches only.

## Continuous Delivery
In order for an architecture such as this to be sustainable in a production environment at scale, it is an absolute necessity that a complete continuous delivery pipeline exists for every component all the way to production. It should be possible for a developer commit to source control to trigger all the necessary testing (fully automated) trigger the requisite approvals and, subject to them being obtained, release the code to the production environment without any human interaction.

## Logging
All components, without exception, should log out to console only. The redirection of logging and its aggregation is handled by the logging solution. Whilst with current technology that will probably mean using solutions such as Splunk or ELK this approach remains open to a better alternative once this is available.

## Centralised Monitoring
This is an area where most HPC applications really have not evolved beyond what was prevalent in the 1990s despite much of the technology industry having moved to far different models. Most of our clients still operate on the basis of checking if a process is running and other simplistic Geneos based validation.

As a part of this architecture however all components must expose performance and operational health metrics to a centralised service (such as Prometheus). These are used not only for checking the health of the production environment but also for (dynamic) capacity planning and even cost allocation in a multi tenanted deployment. This hasn’t been shown on the diagrams in detail simply to allow greater clarity on the application flows however it forms a fundamental part of the architecture.

Hooks to Geneos alerting from a Prometheus/Grafana monitoring stack can be provided to maintain compatibility with existing enterprise monitoring solutions.

## Horizontal Scaling
Whilst this is something that is often already done in most enterprise environments, the scale to which we rely on it is rather different. All processes and services should be single threaded in order to simplify the code with any required scaling being performed through parallel processes rather than multi-threading. 

Additionally, this is no longer something done via a human centric process requiring manual intervention. All parts of the system are designed to scale, through running additional instances, automatically based on the load and the real-time performance metrics.

This implicitly also means that all services/ applications may run multiple instances and as such must be designed as such. Correspondingly, when looking at the architecture diagrams it is implicit that every application or service shown is actually multiple instances.

## High Availability
The “traditional” way to handle high availability for critical applications (in most enterprise deployments), i.e. to the use of dedicated redundant hardware and enterprise grade hardware failure solutions, is not an approach that is suitable either for a containerised architecture or for a system designed to be deployed to public cloud. As such a probability and reliability-based approach is taken to designing resiliency into the system with the necessary level of redundancy being achieved through the duplication of services and applications both within and across data centres.

The uptime requirements and available budget in combination with failure statistics will inform the level of duplication required. The immediate benefit to this approach is that the system is never in a “failed” or “DR” state that must be tested separately. Instead the system is permanently in a state that is regularly run as it auto scales to meet its load requirements.

## FOSS Based
It is no accident that some of the largest technology companies in existence are based almost entirely on FOSS. Open source software acts as a form of leverage that provides the foundations and infrastructure to build the application upon.

Additionally, licensing constraints around the use of third-party software especially licensing costs around many products that are now licensed on a per core or per instance basis make it rapidly prohibitive to design a system that can also be deployed to public cloud.

## Security
Security is a key concern and needs to be an integral part of the design from the outset. Most organisations will have a requirement that all data must be encrypted in transit and at rest and this will need to maintained.
This necessarily mandates the use of encrypted communications between all parts of the system (even on premises). 

Additionally, in an enterprise deployment connectivity to external cloud providers will typically be through the use of dedicated encrypted links only (and we will continue to run application level encryption above this).

## Messaging
In order to facilitate a horizontally scalable design that can be deployed across cloud providers, no messaging technologies should be used. None.

The introduction of a messaging technology is usually done to serve as a buffer in the system and as such introduces an additional stateful component. If the system is designed from the outset such that all state is managed and maintained only in the dedicated caches and transactional boundaries are respected through the use of appropriate API design then in a system that is able to scale all aspects of itself, the need for a secondary buffer in the form of a messaging system disappears.

This results in a simpler overall architecture which in turn also ensures ease of deployment to new environments (on premises or cloud) with minimal effort.

Effectively this means all communication must take place over TCP or UDP based transports.

## Data Movement
The system overall is designed to minimise the movement of data. Micro service-based architecture such as this can be quite chatty and IO heavy. In most modern data centres this is offset by the nature of hardware (30 to 40Mbs aggregated links between machines is fairly standard in a modern cloud provider D/C).

Unfortunately, many on premises enterprise data centres are not quite to the same specification. Our clients will often have single 10Gb/s links to each machine and in some rare cases may be constrained to only 1Gb/s. This is sometimes further compounded with the existence of disparate networks within the enterprise that have limited bandwidth between them.

Additionally, in an architecture that needs to span multiple data centres across different cloud providers we must remain cognisant of the limited network bandwidth to each provider.

As such the general principle adopted is that all messaging will contain only references to where the data may be sourced from. Data will only be sourced at the point where it is required and will be subject to a three-level caching sub system with an LRU eviction policy.

Further, all references to data are immutable. New versions of data must result in new keys.

## Task Duration
HPC workloads vary from Monte Carlo simulations (very large numbers of very short duration tasks) to batch based systems that run a comparatively small number of tasks but where each task will run for much longer periods of time (tens of minutes to hours).

The system must be able to cater for both and as such the latency to start a new task must be very low.

## Multi-Tenanted
The compute farm is envisaged as multi tenanted system with the ability to define discrete pricing profiles, resource limits and preference for each “grid application”.

The schedulers will be able to prioritise and balance load across tenants (if required due to insufficient capacity) based on defined priority levels or ownership of hardware or agreed service levels.

If required tenants may also specify security constraints such as dedicated nodes (to minimise attach vectors based on memory/CPU exploits on the same machine).

## Schedulers and Task Queues and Co-ordinator
In an example of dividing the system down to very small levels of responsibility, the separation of concerns of a task queue (or the co-ordinator) and the schedulers mirrors closely the model used within Kubernetes to schedule workloads.

The task queue is responsible for ordering (on pre-defined priority criteria) the tasks it holds. It has no knowledge of the available worker nodes. The tasks themselves however may have affinity, anti-affinity criteria defined on them, much the same way as pod definition is K8. It may also define which type of task scheduler should be used. If none is specified the default task scheduler is used.

The scheduler is then responsible for the allocation of the task to a specific worker node based on the overall load, the affinity criteria and priority of the task.

Similarly the task co-ordinator performs the same role across clusters instead of within the cluster, moving tasks from one cluster to another to balance load or meet affinity requirements.
