OSG Connect R Tutorial
======================

Overview
--------
This section covers how to use the OASIS system to run a real application like R statistical package. For this example, we'll estimate the value of pi using a Monte Carlo method. We'll first run the program locally, then create a submit file, send it out to OSG-Connect, and collate our results.

Background
----------
Some background is useful here. We define a square inscribed by a unit circle. We randomly sample points, and calculate the ratio of the points outside of the circle to the points inside for the first quadrant. This ratio approaches pi/4.

(See also: http://math.fullerton.edu/mathews/n2003/montecarlopimod.html)

This method converges extremely slowly, which makes it great for a CPU-intensive exercise (but bad for a real estimation!).

Accessing R on the submit host
------------------------------
First we'll need to create a working directory, you can either run {{$ tutorial R}} or type the following:
````
[user@login01 ~]$ mkdir osg-R; cd osg-R
````

First, we'll need to set up the system paths so we can access R correctly. This is done via OSG's [Distributed Environment Modules](https://confluence.grid.iu.edu/display/CON/Distributed+Environment+Modules). To access these modules and access R, enter:
````
[user@login01 osg-R]$ source /cvmfs/oasis.opensciencegrid.org/osg/modules/lmod/5.6.2/init/bash
[user@login01 osg-R]$ module load R
````

Once we have the path set up, we can try to run R. Don't worry if you aren't an R expert, I'm not either.
````
[user@login01 osg-R]$ R
 
R version 3.1.1 (2014-07-10) -- "Sock it to Me"
Copyright (C) 2013 The R Foundation for Statistical Computing
Platform: x86_64-unknown-linux-gnu (64-bit)
R is free software and comes with ABSOLUTELY NO WARRANTY.
You are welcome to redistribute it under certain conditions.
Type 'license()' or 'licence()' for distribution details.
  Natural language support but running in an English locale
R is a collaborative project with many contributors.
Type 'contributors()' for more information and
'citation()' on how to cite R or R packages in publications.
Type 'demo()' for some demos, 'help()' for on-line help, or
'help.start()' for an HTML browser interface to help.
Type 'q()' to quit R.
 
>
````

Great! R works. You can quit out with "q()". 
````
> q()
Save workspace image? [y/n/c]: n
[user@login01 osg-R]$
````

Running R code
--------------

Now that we can run R, let's try using the pi estimation code. Create the file *mcpi.R*:
````R

montecarloPi <- function(trials) {
  count = 0
  for(i in 1:trials) {
    if((runif(1,0,1)^2 + runif(1,0,1)^2)<1) {
      count = count + 1
    }
  }
  return((count*4)/trials)
}
 
montecarloPi(10000000)
````

R normally runs as an interactive shell, but it's easy to run in batch mode too.
````
[user@login01 osg-R]$ Rscript --no-save mcpi.R
[1] 3.141956
````

This should take few seconds to run. Now edit the file. Increasing the trials ten times (10000000) it will take little over a minute to run, but the estimation still isn't very good. Fortunately, this problem is pleasingly parallel since we're just sampling random points. So what do we need to do to run R on the campus grid?

Building the HTCondor job
-------------------------
The first thing we're going to need to do is create a wrapper for our R environment, based on the setup we did in previous sections. Create the file *R-wrapper.sh*:
````bash
#!/bin/bash
 
EXPECTED_ARGS=1
 
if [ $# -ne $EXPECTED_ARGS ]; then
  echo "Usage: R-wrapper.sh file.R"
  exit 1
else
  source /cvmfs/oasis.opensciencegrid.org/osg/palms/setup
  palmsdosetup R
  Rscript $1
fi
````

Notice here that we're using Rscript (equivalent to R --slave). It accepts the script as command line argument, it makes R much less verbose, and it's easier to parse the output later. If you run it at the command line, you should get similar output as above — but you can run this script without first setting up PALMS. This lets the wrapper launch R on any generic worker node under HTCondor.
````
[user@login01 osg-R]$ ./R-wrapper.sh mcpi.R
[1] 3.142524
````

Now that we've created a wrapper, let's build a HTCondor submit file around it. We'll call this one *R.submit*:
````
universe = vanilla
log = log/mcpi.log.$(Cluster).$(Process)
error = log/mcpi.err.$(Cluster).$(Process)
output = log/mcpi.out.$(Cluster).$(Process)
 
# Setup R path, run the mcpi.R script
executable = R-wrapper.sh
transfer_input_files = mcpi.R
arguments = mcpi.R
 
requirements = (HAS_CVMFS_oasis_opensciencegrid_org =?= TRUE)
+ProjectName="ConnectTrain"
queue 100 
````

Notice the requirements line? You'll need to put *HAS_CVMFS_oasis_opensciencegrid_org =?= TRUE* any time you need software from /cvmfs. There's also one small gotcha here – make sure the "log" directory used in the submit file exists before you submit! Else HTCondor will fail because it has nowhere to write the logs.

Submit and analyze
------------------
Finally, submit the job to OSG Connect!
````bash
[user@login01 osg-R]$ condor_submit R.submit
Submitting job(s)....................................................................................................
100 job(s) submitted to cluster 14027.
[user@login01 osg-R]$ condor_q user
 
-- Submitter: login01.osgconnect.net : <128.135.158.173:47839> : login01.osgconnect.net
 ID      OWNER            SUBMITTED     RUN_TIME ST PRI SIZE CMD
14027.0   user           8/25 22:51   0+00:00:43 R  0   0.0  R-wrapper.sh mcpi.
14027.1   user           8/25 22:51   0+00:00:43 R  0   0.0  R-wrapper.sh mcpi.
14027.2   user           8/25 22:51   0+00:00:31 R  0   0.0  R-wrapper.sh mcpi.
14027.4   user           8/25 22:51   0+00:00:41 R  0   0.0  R-wrapper.sh mcpi.
14027.5   user           8/25 22:51   0+00:00:41 R  0   0.0  R-wrapper.sh mcpi.
...
````

You can follow the status of your job cluster with the connect watch command, which shows condor_q output that refreshes each 5 seconds.  Press control-C to stop watching.

Since our jobs just output their results to standard out, we can do the final analysis from the log files. Let's see what one looks like:
````
[user@login01 osg-R]$ cat log/mcpi.out.14027.1
[1] 3.141246
````

After job completion we have 100 Monte Carlo estimates of the value of pi. Taking an average across them all should give us a closer approximation.

We'll use a bit of awk magic to do the averaging:
````
[user@login01 osg-R]$ grep "[1]" log/mcpi.out.* | awk '{sum += $2} END { print "Average =", sum/NR}'
Average = 3.14151
````

That's pretty close! With even more sample sets — that is, more Queue jobs in the cluster — we can statistically come even closer.

What to do next?
----------------
The R.submit file may have included a few lines that you are unfamiliar with.  For example, $(Cluster) and $(Process) are variables that will be replaced with the job's cluster and process id.  This is useful when you have many jobs submitted in the same file.  Each output and error file will be in a separate directory.

Also, did you notice the transfer_input_files line?  This tells HTCondor what files to transfer with the job to the worker node.  You don't have to tell it to transfer the executable, HTCondor is smart enough to know that the job will need that.  But any extra files, such as our MonteCarlo R file, will need to be explicitly listed to be transferred with the job.  You can use transfer_input_files for input data to the job, as shown in Transferring data with HTCondor.
