---
layout: lesson
root: ../..
title: Data storage and transfer
---
<div class="objectives" markdown="1">

#### Objectives
*   Discover how to transfer input and output data  
</div>


<h2> Overview </h2>

In this lesson, we will learn the basics of data storage and transfer for DHTC jobs and on OSG Connect. 

<h2>Introduction to Data Handling for DHTC jobs</h2>

Data handling is one of the trickiest parts of effectively using DHTC computing.
New users often run into issues in regards to effectively transferring inputs to compute
jobs or with getting output back.  However, most situations can be broken down
into a few basic use cases: 

*   Jobs with small inputs and outputs
*   Jobs with large inputs but small outputs
*   Jobs with small inputs but large outputs
*   Jobs with both large inputs and large outputs

We will consider each use case one by one.

<h3>Jobs with small inputs or outputs</h3>

Jobs without large inputs or outputs are the easiest case to handle.   If jobs
have inputs and outputs that are smaller than ~1 gigabyte then the builtin HTCondor
file transfer mechanims will work to handle transfers to and from the compute node.  
Input files can be specified using the `transfer_input_files` option in your
submit file.  Likewise output files and directions can be specified using
`transfer_output_files`.  Once the input and output files are given to HTCondor,
it will automatically transfer the input files to compute nodes when jobs are
scheduled to run and then transfer the output after the job completes back to
your submit node.

<h3>Jobs with large inputs but small outputs</h3>

Jobs with large inputs but small outputs are a little harder to handle.
Examples of these types of jobs, would be things such as BLAST which does
searches for similar DNA sequences.  The inputs (DNA reference databases and DNA
sequence of interest) can easily be tens or hundreds gigabytes.  However, the
output (locations of similar sequences and similarity of these sequences) tends
to be kilobytes or megabytes in size.  

If the majority of the inputs are databases or reference files a possible
solution would be to place the input files on a publicly accessible webserfer.
Then jobs would wget the appropriate files when they start running.  Since most
OSG sites have a Squid server, this will minimize the network traffic outside of
the site running your jobs.  If the majority of the input is in reference files
that are identical between jobs, then another alternative would be to use a
service such as OASIS that will make the reference files available on the
majority of OSG sites.  Jobs can then access the files as if they are on the
compute node and only the files that are accessed will be transferred.  If the
input files need to remain private or if the inputs drastically change between
jobs, you should probably discuss this with us so that we can help you come up
with a good solution.

Since the output files are small (i.e. < ~1GB), using the `transfer_output_files` option
in your submit file and allowing HTCondor to manage transferring outputs from
the compute nodes to the submit node should work without problems.

<h3>Jobs with small inputs and large outputs</h3>

This use case deals with jobs that generate large outputs from a small set of
inputs.  For example, a simulation may generate a large data set showing the
evolution of a physical system based on a small set of input parameters.  

The inputs for this type of jobs can be handled using the HTCondor transfer
mechanisms.  By using the `transfer_input_files` option in your submit file,
HTCondor will automatically handle transferring inputs to compute nodes when
your job runs.  

The outputs for this type of job are more difficult to handle.  Solutions may
range from compressing the outputs to generate smaller, more manageable files to
setting up or utilizing a service like gridftp to copy files back to your login
node or a storage system for later retrieval.  Regardless, it is probably best
to talk to us so that we can come up with the best solution for your transfer
needs. 

<h3>Jobs with both large inputs and large outputs</h3>

This use case is essentially just a combination of the prior two use cases.  As
such, inputs can be handled using a web server or OASIS and transferring outputs
will probably need some discussion with us to determine the best solution.

<h3>Guidelines for Data Storage</h3>

|   | **Recommended Limit**| **Purpose** | **Network mounted** | **Backed Up** | **Quota** | **Purge** | **Details**|
|:------- |:----------------:|:------|:------:|:------:|:------:|:------:|:----------|
| **home**    |  < 20 GB      | Meant for quick data access and not for submitting jobs.| Yes | Yes | 20 GB | No | [Data Storage](https://support.opensciencegrid.org/support/solutions/articles/12000002985-storage-solutions-on-osg-home-stash-and-public)|
| **local-scratch**   |  < 25 GB      | Meant for large temporary storage and I/O for your jobs. Files older than 30 days are automatically removed. | No | No | No | Yes, 30 days | [Data Storage](https://support.opensciencegrid.org/support/solutions/articles/12000002985-storage-solutions-on-osg-home-stash-and-public)|
| **stash**   |  N/A          | Meant for large storage and I/O for your jobs. | Yes | No | No | no | [Data Storage](https://support.opensciencegrid.org/support/solutions/articles/12000002985-storage-solutions-on-osg-home-stash-and-public)|
| **public**  |  N/A          | Meant for sharing data and transfer input data via HTTP or staschcp | Yes | No | No | No | [Data Storage](https://support.opensciencegrid.org/support/solutions/articles/12000002985-storage-solutions-on-osg-home-stash-and-public)|

<h2>Exploring the Stash system</h2>

First, we'll look at accessing Stash from the login node.  You'll need to log in to OSG Connect:

~~~
ssh username@login.osgconnect.net #Connect to the login node with your username
passwd:       # your password
~~~

Once done, you can change to the `stash` directory in your home area:

~~~
$ cd ~/stash    
~~~

This directory is an area on Stash that you can use to store files and
directories.  It functions just like any other UNIX directory although it has
additional functions that we'll go over shortly.

For future use, let's create a file in the public part of Stash:

~~~
$ cd ~/stash/public/
$ echo "Hello world" > my_hello_world
~~~

In addition, let's create a directory as well for future use:

~~~
$ mkdir my_directory
~~~


<h2>Transferring files to and from Stash using SCP </h2> 

We can transfer files to Stash using `scp`.  `Scp` is a counterpart to ssh that allows for
secure, encrypted file transfers between systems using your ssh credentials.    

To transfer a file from Stash using `scp`, you'll need to run `scp` with the
source and destination.  Files on remote systems are indicated using
user@machine:/path/to/file .  Let's copy the file we just created from Stash to
our local system:

~~~
$ scp username@login.osgconnect.net:~/stash/public/my_hello_world .
~~~

As you can see, `scp` uses similar syntax to the `cp` command that you were shown
previously.  To copy directories using `scp`, you'll just pass the `-r` option to
it.  E.g:

~~~
$ scp -r username@login.osgconnect.net:~/stash/public/my_directory .
~~~

> #### Challenges
>
> * Create a directory with a file called `hello_world_2` in the `~/stash/public` directory and copy it from Stash to your local system.
> * Create a directory called `hello_world_3` on your local system and copy it to the `stash` directory.

<h2>Stash, staschp and stashcache</h2>

StashCache is a data service for OSG which transparently caches data
near compute sites for faster delivery to active grid jobs. The standard
practice of using HTCondor file transfer or http to move data files to
grid sites can be inefficient for certain workflows. StashCache uses a
distributed network filesystem (based on XRootD proxy caching) which
provides an alternative method for transferring input files to compute
sites. It is implemented through a handful of strategically-distributed
sites which provide caching of the input data files.

<h3>When to use StashCache</h3>

StashCache typically outperforms other methods in the following cases:

* Jobs require large data files (> 500 MB)
* The same data files are used repeatedly for many separate jobs

<h3>How to use StashCache</h3>

1)  Include the following line in the job's submit script to indicate that StashCache is required:

    +WantsStashCache = true

2)  Use the 'stashcp' command-line tool in the job wrapper script to transfer necessary data files to the compute host.

First load the stashcp module:

~~~
$ module load stashcp
~~~

Then transfer your data:

~~~
$ cd $HOME
$ stashcp /user/<userid>/public/my_hello_world $HOME/
~~~


<h2>StashCache Blast</h2>





<h2>Transferring files to and from Stash using Globus</h2>

An alternate method for accessing Stash is to use Globus.  Globus allows you
to initiate transfers between Globus endpoints and will handle the actual file
and directory transfers transparently without needing further input from you.
When the transfer is complete, Globus will send a notification to you indicating
this.

Let's transfer a file from your laptop to Globus.  Doing this will require an 
application called Globus Connect Personal that allows Globus to transfer files
and directories to and from your laptop. First go to the [Globus
page](https://www.globus.org/globus-connect-personal) and download and install
the globus connect personal installer specific to your system.  

While that's running, you'll need to get a setup key from Globus in order to
setup the Globus Connect Personal software.  

*   First go to the [Globus](http://globus.org) page and register or login
*   Next go to this [page](https://globus.org/xfer/ManageEndpoints#)
*   Click on the add Globus Connect Personal link
*   Enter a name for your endpoint on the page (remember this!)
*   Click on "Generate setup Key"
*   Select the key and copy the key

Finally, start the Globus online personal software that you just installed.  The
installer will ask for the setup key that you obtained from the Globus website.
At this point, the install and setup of Globus Connect Personal is complete.

Now go to [Globus](https://www.globus.org/app/transfer). 
For the first endpoint, enter username#name
where name is the name you choose for the endpoint above. You should now see the
files from your laptop displayed.  For the second endpoint, enter
`connect#stash` and hit enter.  You should now see the contents of your home
directory on OSG Connect.  Now double click on the `data` directory.  Select a
file on your laptop and click on the right arrow on the top of the screen to
start a transfer to Stash. You can transfer files or directories to your
laptop by selecting it in the Stash window and selecting the left arrow.

> #### Challenges 
>
> * Copy a file to Stash from your laptop using Globus.  
> * Next copy the `my_hello_world` file from Stash to your laptop using Globus.

<h2>Putting it all together</h2>

Let us do an example calculation to give us a taste of a what a typical
job may involve. We will peform a  molecular dynamics simulation of a
small protein in implicit water. To get the necessary files, we use the *tutorial*
command on OSG. 

Log in to OSG Connect:

~~~
$ ssh username@login.osgconnect.net
~~~

Type:

~~~
$ tutorial namd
$ cd ~/tutorial-namd
$ rm *.inp
$ rm *.conf
~~~

*Aside*: [NAMD](http://www.ks.uiuc.edu/Research/namd/) is a widely used
molecular dynamics simulation program. It lets users specify a molecule in some
initial state and then observe its time evolution subject to forces.
Essentially, it lets you go from a specifed molecular
[structure](http://en.wikipedia.org/wiki/Superoxide_dismutase#mediaviewer/File:Superoxide_dismutase_2_PDB_1VAR.png)
to a [simulation](https://www.youtube.com/watch?v=mk3cLd9PUPA&list=PL418E1C62DD9FC8BA&index=1)
of its behavior in a particular environment.  It has been used to study polio
eradication, similations of graphene, and studies of biofuels.

You should see the following files in the directory:

~~~
$ ls
namd_run.sh  namd_run.submit     README.md
ubq.pdb      ubq.psf
~~~

The files 

~~~
namd_run.submit #HTCondor job submission script file.
namd_run.sh #Job execution script file.
ubq.pdb #Input pdb file for NAMD.
ubq.psf #Input file for NAMD.
~~~

We need two files in order to successfully run the simulation. 
The file `par_all27_prot_lipid.inp` which is the parameter file and is required for 
the NAMD simulations. The other file needed is `ubq_gbis_eq.conf` which has
the input configuration for the NAMD simulations.  

> #### Challenges
>
> * You can find the missing files at [here](https://stash.osgconnect.net/+sthapa/tutorial-namd), 
    download the files from that website and upload it to the tutorial-namd file on OSG Connect.

Now submit the NAMD job. 

~~~
$ condor_submit namd_run.submit 
~~~

Once the job completes, you will see non-empty `ubq_gbis_eq.0.log`
file where the standout output from the programs is written.

> #### Challenges
>
> * Download the output file from the NAMD job to your laptop and view it.  Try
> using a different method to transfer the file than before.


You should see:


~~~
$ tail  ubq_gbis_eq.0.log

WallClock: 6.084453  CPUTime: 6.084453  Memory: 53.500000 MB
Program finished.
 
~~~

The above lines indicate the NAMD simulation was successful. 

<div class="keypoints" markdown="1">

#### Key Points
* Data can be transferred in and out of Stash using scp and Globus
</div>

