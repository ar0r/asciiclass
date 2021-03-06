# Hadoop: the next generation

Carlo Curino: Resource allocation frameworks

The origin

  - Google led the pack w/ Google Filesystem + MapReduce (2003/2004)---general-purpose systems-building that helps whatever tools you're building scale up and be fault-tolerant.
  - Open-source and parallel efforts: Yahoo! Hadoop ecosystem: Hadoop file system + Hadoop MapReduce (2006/2007)
   -> Now Twitter, LinkedIn, and a number of companies add to this effort
  - Microsoft releases programming model (Scope/Cosmos) that makes it easier to build data workflow than Map + Reduce (2008)
  * open sourced to compete with market leaders (google)


Why did Hadoop grow so fast?

 - all of the data is in one location (HDFS).  this includes other groups' previously-hard-to-access data.
 - anyone can run anything on hundreds of machines without getting permission to access some five-node database.
 - Insights from raw data are cool, but it also was the case that everyone started thinking that "Big Data" == "Hadoop"
 * Access is huge
    * access to data
    * access to compute power
    * access to fault tolerance computing
    * lesson == paperwork really sucks 

Problem: everything has to be cast as MapReduce

  - Has to be a map-only job: graph processing?  iterative algorithms?  stream processing?  ugh.
  - What if I want to run web servers!?  Map task!
  - Great Carlo story: someone just used mapreduce as a way to get tens of machines doing graph processing for him, completely ignoring mapreduce input, and just keeping the process running

Another problem: small companies/startups are now running computation on small clusters (10 machines) or public clouds

  - Smaller data, less machine failures: no longer need MapReduce's fault tolerance.  Efficiency matters more than scalability.
  - No longer have to worry about 1000-way communication between machines across switches/racks in a datacenter
  - Administration/tuning is done by mere mortals, not a trained administrator

Another problem: running inside a public cloud like amazon web servies

  - Security: Untrusted users (security)
  - Guarantees: Users pay per machine, so availability/reliability trade off with that
  - Resource sharing: Users are unrelated to one-another, but their workloads might conflict with one-another

Since you're running multiple jobs from different users, you have a resource allocation problem

  - Keep computation as close as possible to the data: run Map() on the same node or rack as the data
  - Keep jobs that compete for the same resource from being collocated: high disk utilization vs. high cpu utilization
  - Some users might have higher priority to run---make sure their stuff runs with priority but don't starve out the other users.

Old Hadoop 1.0 Architecture

  - One jobtracker gets requests for jobs, coordinates map and reduce tasks
  - As mapreduce requests come in, allocates them to new map/reduce nodes
  - Problems:
    - jobtracker is a single point of failure---all jobs fail if it dies, and no new tasks can start until you restart the task
    - jobtracker has scalability limitations---only one tracker manages entire cluster
    - jobtracker has two tasks: managing resources and managing application workflow
    - there are fixed slots for map tasks and reduce tasks.  what if you have more maps than reduces?  no luck!
    - slow maps and reduces hold up other maps and reduces
    - can only run mapreduce on this cluster: what about streaming or graph processing?

Hadoop 1.0 has so many problems, how about buling a clusterOS focused on separating resource management from the "map reduce" type applications?


Three proposals to get a better version

  - YARN (Yet Another Resource Negotiator)---Hadoop 2.x, production at Yahoo!, released into general availability yesterday
  - Mesos (2011, from Berkeley, open-sourced and running at Twitter)
  - Omega (2013, Google paper)
  
What are applications that would run on a clusterOS?

* Giraph
* MR
* Storm (stream processing from twitter)
* Dryad (workflow system)
* Reef (light weight containerization.  Think of JVM as an application specific "VM")
* Your own thang  


### Yarn


What is it? 

  - New version of HDFS, but not much to write home about
  - YARN offers operating system-like API on top of HDFS: get access to resources and run whatever you want on them.
  - MapReduce 2.0 runs on top of YARN, but you can also run Storm (streaming), Giraph (graph processing), or even ad-hoc things like web servers.
  - How running a new job works
    - A client submits a job to YARN.  A resource manager assigns you machines for it, and on the first machine, runs an Application Master for you that does the rest of your resource management.
    - The App Master calls the central resource manager back and asks for resources (5 containers, 50 gigs of ram, 10 processors, 1 TB disk, ideally on two machines, hopefully in same rack, but it's OK if they are in the same data center).
      -> e.g., the mapreduce app master says: I want to run 500 mappers, so let's ask for 500 containers with 1 GB memory and 5 GB disk each.  My data is on rack 10, so try to put my containers near there.  Now it's time to run reducers: I need 10 of those, and can start shutting down mappers as they complete.
    - Each job you run gets one app master.  For example, two mapreduce jobs will get two application masters.  This makes it easy to upgrade software, etc.


Components

* Resource manager
* Scheduler: makes sure there is fair resource scheduling 
* App Master is a specialized manager for an instance of your YARN application (e.g., spark app master).  
    * An instance runs for every job
    * think query optimizer if you have a DB application
* Tasks are controlled by the appmaster

Resource manager is decoupled from running the applications

App Master can request more resources after the fact.  If this is the case, we need a couple things:

* an abstraction for a bundle of resources (2GB ram, 25% of disk throughput, 3 cores, etc…) 
* Ability to move tasks around
    * Need to save and load task state (so system can move container to another machine)

### Mesos

  - Allows you to run multiple frameworks (Spark, MapReduce, Storm, etc.) on the same cluster.
  - Rather than one app master per job, it runs one resource manager per framework.
    * allows optimization across framework specific jobs
    * more complex to write a scheduler
    * Mesos needs less information about the frameworks/jobs  
  - A Mesos scheduler round-robins between the resource managers, ensuring they have the most resources available for them at the same time.
  - Benefits: easier to run things like long-running services (e.g., GMail) on top of it---doesn't assume you're running computation frameworks.
* locks resources for each framework
  


### Omega

  - Rather than having a single scheduler, you give the global cluster state to each framework.
  - Frameworks respond to you and ask you to allocate parts of this cluster to them.  Resolve conflicts on their behalf---wasn't sure how.
  
  

### Huge benefit of these systems:

  - helps people innovate
  - used to be: i improve mapreduce, and Yahoo! has to take down the cluster and update it.
  - now: run a new application container with my software, compare it side-by-side.