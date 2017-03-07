---
layout: lesson
root: ../..
title: Workflow management with DAGMan
---

<div class="objectives" markdown="1">
#### Objectives
* Learn about graphs as they relate to computation
* Learn how a graph manager can implement a workflow management system
* Use DAGMan to manage a set of molecular dyanmics calculations
* How to write a DAGMan input file for a workflow
</div>



<h2> Overview </h2>

In High Throughput Computing, one is typically faced with having to manage a large set of computational tasks. This can include situations in which tasks may be depend on one another. Workflow management systems can help relieve job management burden for the user. [DAGMan](http://research.cs.wisc.edu/htcondor/dagman/dagman.html) (Directed Acyclic Graph Manager) is a workflow management system based on graphs (see figures below) build into HTCondor. DAGMan handles sets of computational jobs 
that can be described as a set of nodes in a "directed acyclic graph". This means that the job dependencies do not form a loop, see "Cyclic" vs. "Acyclic" figure below. 

![fig 1](https://raw.githubusercontent.com/OSGConnect/tutorial-dagman-namd/master/DAGManImages/Slide1.png)

In this tutorial we will learn how to apply DAGMan to help manage a set of molecular dynamics (_MD_) simulations using the [NAMD](http://www.ks.uiuc.edu/Research/namd/) program. NAMD is conventionally used in highly parallel HPC settings, scaling to thousands of cores managed by a single job. One can achieve the same scaling and ease of management in [HTC systems](http://en.wikipedia.org/wiki/High-throughput_computing) using thousands of individual jobs using workflow tools such as DAGMan. 

![fig 2](https://raw.githubusercontent.com/OSGConnect/tutorial-dagman-namd/master/DAGManImages/Slide5.png)


<h2> Running Jobs with DAGMan </h2>  

There are several different reasons why one might chose to use DAGman to submit and manage HTCondor jobs rather than, for example, the `Queue` command. At present, the recommended execution time for a job on OSG is 2-3 hours or less. Jobs requiring more than 2-3 hours may be terminated before completition. Managing the resubmission of prematurely terminated jobs manually is not practical. DAGMan offers an elegant and simple solution to reduce the management burden of. Similarly, one may have to manage a series of jobs that are interdependent in some way. An example of job interpendency is that they require a previous task (or set of tasks) to complete before they can be executed.

<h3> No dependency DAG </h3>

The first example will be a DAG that does not have any job interdependencies. This situation arises when the same computation or task is supposed to be performed on a set of input files, or when the output of each computation or task is independent. An example of such a situation is applying cuts to a dataset that consists of several or many input files.

The DAGMan script and the necessary files are available to the user by invoking the `tutorial` command. 

    $ tutorial dagman-namd
    $ cd tutorial-dagman-namd/NoDependencyDAG

The directory `tutorial-dagman-namd/NoDependencyDAG` contains all the necessary files. The file `nodependency.dag` is the DAGMan input file. The files `namd_run_job0.submit`, etc. are the HTCondor job description files that execute the script `namd_run_job0.sh`, etc.

Let us take a look at the DAG file `nodependency.dag`.  

    $ cat nodependency.dag 

    ######DAG file###### 
    ##### Define Jobs ###
    #####  JOB JobName JobDescriptionFile
    JOB A0 namd_run_job0.submit
    JOB A1 namd_run_job1.submit
    JOB A2 namd_run_job2.submit
    JOB A3 namd_run_job3.submit

To define a job (or node in DAG lingo), a line begins with `JOB` followed by a unique identifier for that job, for example, `A0` for the first line and, the HTCondor submit file to be used, i.e. `namd_run_job0.submit`.  A line begins with pound (#) character is a comment. 

We submit the DAGMan task using the command `condor_submit_dag` 

    $ condor_submit_dag nodependency.dag 

    -----------------------------------------------------------------------
    File for submitting this DAG to Condor           : nodependency.dag.condor.sub
    Log of DAGMan debugging messages                 : nodependency.dag.dagman.out
    Log of Condor library output                     : nodependency.dag.lib.out
    Log of Condor library error messages             : nodependency.dag.lib.err
    Log of the life of condor_dagman itself          : nodependency.dag.dagman.log
    
    Submitting job(s).
    1 job(s) submitted to cluster 1317501.
    -----------------------------------------------------------------------
    

Let's monitor the job status every two seconds. (Recall `connect watch` from a previous lesson.

    $ connect watch 2

    -- Submitter: login01.osgconnect.net : <192.170.227.195:48781> : login01.osgconnect.net
    ID      OWNER            SUBMITTED     RUN_TIME ST PRI SIZE CMD               
    1317646.0   username          10/30 17:27   0+00:00:28 R  0   0.3  condor_dagman     
    1317647.0   username          10/30 17:28   0+00:00:00 I  0   0.0  namd_run_job0.sh
    1317647.0   username          10/30 17:28   0+00:00:00 I  0   0.0  namd_run_job1.sh
    1317647.0   username          10/30 17:28   0+00:00:00 I  0   0.0  namd_run_job2.sh
    1317647.0   username          10/30 17:28   0+00:00:00 I  0   0.0  namd_run_job3.sh

    4 jobs; 0 completed, 0 removed, 4 idle, 1 running, 0 held, 0 suspended

We need to type `Ctrl-C` to exit from watch command. We see five running jobs. One is the DAGMan job which manages the execution of jobs inside the DAG. The others are the jobs controlled by the DAG. Once the DAG completes, you will see four `.tar.gz` files `OutFilesFromNAMD_job0.tar.gz`, `OutFilesFromNAMD_job1.tar.gz`, `OutFilesFromNAMD_job2.tar.gz`, and `OutFilesFromNAMD_job3.tar.gz`. If the output files are not empty, the jobs are successfully completed. Of course, a thorough check requires inspection of the results.  

<h3> Linear DAG </h3>

In our second example, we will use four MD simulation that depend linearly on each other. This means that will be submitted sequentially by HTCondor. For the sake of simplicity, we have artifically reduced the number of computation steps for each simulation.

![fig 3](https://raw.githubusercontent.com/OSGConnect/tutorial-dagman-namd/master/DAGManImages/Slide2.png)

Instead of running the four jobs from the above example independently, we want to run them sequentially, i.e. `A0-->A1-->A2-->A3`. In these calculations, the output files from the job `A0` serves as an input for the job `A1` and so forth. These set of jobs clearly represents an acyclic graph. In DAGMan language, job `A0` is parent of job `A1`, job `A1` is parent of `A2` and job `A2` is parent of `A3`. 

First:

    $ cd ../LinearDAG

Let us take a look at the DAG file `linear.dag`.  

    $ cat linear.dag 

    ######DAG file###### 
    ##### Define Jobs ###
    #####  JOB JobName JobDescriptionFile
    JOB A0 namd_run_job0.submit
    JOB A1 namd_run_job1.submit
    JOB A2 namd_run_job2.submit
    JOB A3 namd_run_job3.submit

    ##### Relationship between Jobs ###
    ##### PARENT JobName CHILD JobName
    PARENT A0 CHILD A1
    PARENT A1 CHILD A2
    PARENT A2 CHILD A3

The file looks similar to the one used for the no dependency example above. The main addition are the last three lines containing the definition of the job interdependency. The `PARENT` and `CHILD` commands describes the dependency between jobs, where a job(s) following the `PARENT` command need to successfully complete before jobs following `CHILD` command are submitted. 

If we now submit the DAG:

    $ condor_submit_dag linear.dag 

    -----------------------------------------------------------------------
    File for submitting this DAG to Condor           : nodependency.dag.condor.sub
    Log of DAGMan debugging messages                 : nodependency.dag.dagman.out
    Log of Condor library output                     : nodependency.dag.lib.out
    Log of Condor library error messages             : nodependency.dag.lib.err
    Log of the life of condor_dagman itself          : nodependency.dag.dagman.log
    
    Submitting job(s).
    1 job(s) submitted to cluster 1317501.
    -----------------------------------------------------------------------
    

Let's monitor the job status every two seconds. (Recall `connect watch` from a previous lesson.)

    $ connect watch 2

    -- Submitter: login01.osgconnect.net : <192.170.227.195:48781> : login01.osgconnect.net
    ID      OWNER            SUBMITTED     RUN_TIME ST PRI SIZE CMD               
    1317646.0   username          10/30 17:27   0+00:00:28 R  0   0.3  condor_dagman     
    1317647.0   username          10/30 17:28   0+00:00:00 I  0   0.0  namd_run_job0.sh

    2 jobs; 0 completed, 0 removed, 4 idle, 1 running, 0 held, 0 suspended

Only two jobs are running now: the DAGMan job and the top-level parent, i.e. job `A0`. `A1` through `A3` will have to wait until their parent job(s) have completed before the are executed.

<h3> PRE and POST processing of jobs </h3>

![fig 3a](https://raw.githubusercontent.com/OSGConnect/tutorial-dagman-namd/master/DAGManImages/Slide07.png)

Sometimes, we need to perform a task before a job is submitted or after it is completed. Such pre-processing and post-processing are handled in DAGMan via SCRIPT command. Now let us see how this work for the linear DAG of NAMD jobs. 

    $ cd LinearDAG_PrePost
    $ cat linear_prepost.dag

    ######DAG file###### 
    ##### Define Jobs ###
    #####  JOB JobName JobDescriptionFile
    JOB A0 namd_run_job0.submit
    JOB A1 namd_run_job1.submit
    JOB A2 namd_run_job2.submit
    JOB A3 namd_run_job3.submit

    ##### Relationship between Jobs ###
    ##### PARENT JobName CHILD JobName
    PARENT A0 CHILD A1
    PARENT A1 CHILD A2
    PARENT A2 CHILD A3

    ##### PRE or POST processing of a job
    ##### SCRIPT PRE/POST JobName ProcessScript
    SCRIPT PRE   A0  pre-script-temperature.sh
    SCRIPT POST  A3  post-script-energy.sh

Except the last four lines block, this DAG file `linear-post.dag` is same as the previous DAG file `linear.dag`. The script block specifies a pre script and/or post script and which job they are associated with. 

The pre script `pre-script-temperature.sh` sets the temperature for the simulations and it is processed before the job `A0`.  This means the pre script is the first thing processed before any job is submitted.  The post script `post-script-energy.sh` runs after finishing all the simulation jobs `A0`, `A1`, `A2`, and `A3`. It extracts the energy values from the simulation results.  Both pre and post scripts are executed on the local submit machines and not on a remote worker. These scripts should be as lightweight processes as possible.

<h3> Parallel DAG </h3>

![fig 4](https://raw.githubusercontent.com/OSGConnect/tutorial-dagman-namd/master/DAGManImages/Slide3.png)

Now we consider the workflow of two-linear set of jobs `A0`, `A1`, `B0` and `B1`. Again these are NAMD jobs. The job `A0` is parent of `A1` and the job `B0` is the parent of `B1`. The jobs `A0` and `A1` do not depend on `B0` and/or `B1`. This means we have two parallel workflows that are represented as `A0->A1` and `B0->B1`. The arrow shows the order of job execution. This example is located at 

	$ cd tutorial-dagman-namd/TwoLinearDAG

The directory contains the input files, job submission files and execution scripts.  What is missing here is the `.dag` file. See if you can write the DAGfile for this example and submit the job. 

<h3> X-DAG </h3>
We consider one more example workflow that allows the cross communication between two parallel pipelines. The jobs `A0` and `B0` are two independent NAMD simulations. After finishing `A0` and `B0`, we do some analysis with the job `X`. The jobs `A1` and `B1` are two MD simulations independent of each other. The `X` job determines what is the simulation temperature of MD simulations `A1` and `B1`. In DAGMan lingo, `X` is the parent of `A1` and `B1`.  

![fig 5](https://raw.githubusercontent.com/OSGConnect/tutorial-dagman-namd/master/DAGManImages/Slide4.png)

The input files, job submission files and execution scripts of the jobs are located in the `X-DAG` subdirectory:

	$ cd tutorial-dagman-namd/X-DAG

Again we are missing the `.dag` file here. See if you can write the DAG file for this example. 

<h2> Job Retry and Rescue </h2>

In the above examples, the set of jobs have simple inter relationship. Indeed, DAGMan is capable of dealing with set of jobs with complex interdependencies. One may also write a DAG file for set of DAG files where each of the DAG file contains the workflow for set of condor jobs. Also DAGMan can help with the resubmission of uncompleted portions of a DAG, when one or more nodes result in failure.  

<h3> Job Retry </h3>

Say for example, job `A2` in the above example is important and you want to eliminate the possibility as much as possible. One way is to retry the specific job `A2` a few times. DAGMan would retry failed jobs when you specify the following line at the end of dag file:

	$ nano linear.dag # Open the linear.dag file
	 

At the end of the linear.dag file
	 
	Retry A2 3 #This means re-try job A2 for three times in case of failures. 
	

If you want to retry jobs A2 and A3 for 7 times,  edit the linear.dag 
	 
	### At the end of the linear.dag file
	Retry A2 7 #This means re-try job A2 for up to seven times in case of failures.
	Retry A3 7 #This means re-try job A3 for up to seven times in case of failures.
 
 <h3> Rescue DAG </h3>

If DAGMan fails to complete the complete task, it creates a rescue DAG file with a suffix `.rescueXXX`, where `XXX` is a number starting at `001`. The rescue DAG file contains the information about where to restart the jobs. Say for example, in our workflow of four linear jobs, the jobs `A0` and `A1` are finished and `A2` is incomplete. In such a case we do not want to start executing the jobs all over again but rather we want to start from Job `A2`. This information is embedded in the rescue DAG file. In our example of `linear.dag`, the rescue DAG file would be `linear.dag`. So we re-submit the rescue DAG task as follows:

	$ condor_submit_dag linear.dag
 
<div class="keypoints" markdown="1">
#### Key Points
* DAGMan handles computational jobs that are mapped as a directed acyclic graph.
* `condor_submit_dag` is the command to submit a DAGMan task. 
*  Job Retry and Job Rescue mechanism in DAGMan are helpful when running a complex workflow
</div>


