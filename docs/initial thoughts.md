# Running HPC Workloads on Kubernetes

Pardon the vagaries here but this is a work in progress at this stage. The intention is very much that although this might be something written on top of K8 in the initial stages that some of these constructs might become native Kubernetes objects should the desire to run HPC on K8 be sufficiently prevalent.

## TL;DR
Run HPC tasks on long running services in Kubernetes pods and have a queue of HPC tasks with the number of pods being scaled through use of auto horizontal scaling based off custom metrics in Prometheus. The use of long running services rather than Jobs in Kubernetes is deliberate to allow efficient utilisation for very short duration tasks that are often prevalent in HPC workloads (especially Monte Carlo simulations).

## The Long Version
So, we start with an HPC Task object which might look something like this:
```
HPCTask
--------
Name
Type
Version
Parameters / payload
ID
Task to Image Mapper
```
These would be created by the client and added to the list. In a similar fashion to how the client doesn’t allocate a pod to a node directly (but can influence that choice) a client doesn’t select which containerised image should execute the task. Instead a Task to Image mapping process (a default one or a specific one specified by the task) selects the most appropriate image to use to execute the task.
```
HPCTaskRunnable
---------------
HPCTask
Image
```
Once an execution image has been assigned to a task we have a list of executable tasks. Note, that the image in this object is the image that will process the task itself.

This list is subscribed to by another process which then annotates each task with the estimated execution time of each task. This is then exposed into Prometheus via the Custom Metrics API.

In addition, this list of executable tasks is subscribed to by a process responsible for the construction of the pods that will run the tasks. It monitors the list of runnable tasks to construct an outstanding list of images that are currently required. If it is determined that a suitable pod doesn’t exist to run the task then the Task Pod Creator will create a new Pod that contains an image which is responsible for communicating with the HPC task list, HPCTaskClient, and also the image required to process the task.

The image required to process the task must expose an HTTP API (visible within the pod only) to allow the task client to control the execution if the task. The API might look a little like

```
initialize
execute(parameters / payload)
finalize
ping
```

The initialize and finalize methods are optional but the execute method must be implemented and should execute the task. The payload argument is the payload defined in the HPC task itself.

The ping method is used by the client to ensure that the process is still alive. Conversely the Task Client exposes the following API

```
complete(status)
```

which should be called by the image upon completion of the task. Possible status codes are completed, failed, killed.

The Kubernetes auto horizontal scaling is then configured to auto scale based on the total execution time of tasks outstanding on the queue for each image/pod required.

This gives the basics of execution of HPC workloads however is missing the basic requirements for a multi tenanted grid. Additional constructs around limiting pod scaling, assigning resource limits and measuring utilisation would be required for this to be a viable HPC solution.
