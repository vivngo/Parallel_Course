Hands-on lab (Second Part)
================
Sofia Gil
January 15, 2019

-   [First steps for parallelizing in R](#first-steps-for-parallelizing-in-r)
    -   [The basics](#the-basics)
    -   [Divide and Conquer](#divide-and-conquer)
-   [Submitting R scripts into the cluster](#submitting-r-scripts-into-the-cluster)
    -   [The basics](#the-basics-1)
    -   [How does the cluster work?](#how-does-the-cluster-work)
    -   [Non-parallel jobs](#non-parallel-jobs)
    -   [Parallel jobs](#parallel-jobs)
    -   [Other important LSF commands](#other-important-lsf-commands)
-   [References](#references)

First steps for parallelizing in R
==================================

The basics
----------

It is well known that when we are using *R* we always have to avoid *f**o**r* loops. Due to the calls that *R* does to *C* functions, therefore it is better to vectorize everything.

``` r

rm(list=ls())
gc()
#>          used (Mb) gc trigger (Mb) max used (Mb)
#> Ncells 553917 29.6     940480 50.3   750400 40.1
#> Vcells 945774  7.3    1650153 12.6  1203988  9.2

len<-5000000

a<-runif(len)
b<-runif(len)

c1=0

system.time({
  for(i in 1:length(a)){
    c1<-a[i]*b[i]+c1
  }
})['elapsed']
#> elapsed 
#>    0.28

system.time({c2<-a%*%b})['elapsed']
#> elapsed 
#>    0.04
```

But... what should we do when the problem we are facing needs a loop? There are two options:

1.  Write the loop and wait.(*Easy way*)
2.  Paralellize. (*Fun way*)

Imagine we want to do the Kronecker Product of *A* ⊗ *B*, that means:

<img src="Github_files/figure-markdown_github/Kronecker.png" width="300" />

#### Easy way

**Memory access:** It is important to know the way *R* stores matrices, i.e., by row or by column. In the specific case of *R* all the matrices are stored by column, given that *R* is a statistical software where the columns almost always represent variable names. This is crucial since each variable created in R has an ID number, so the compiler or the interpreter will assign specific addresses in memory at which every element of the variables (e.g. vector, matrices, data frames, lists,...) will be stored. Given that these addresses are consecutive, it will always decreases the memory access time if we call them in a consecutive way.

<img src="Github_files/figure-markdown_github/ColumnOriented.png" width="300" />

``` r

(A<-matrix(runif(9),nrow = 3,ncol = 3))
#>           [,1]      [,2]      [,3]
#> [1,] 0.4809838 0.4209527 0.5386734
#> [2,] 0.1871779 0.6487687 0.2398545
#> [3,] 0.7857373 0.2676853 0.1332319

for(i in 1:length(A))
  print(paste(i,A[i]))
#> [1] "1 0.48098376346752"
#> [1] "2 0.187177889514714"
#> [1] "3 0.785737308906391"
#> [1] "4 0.420952731743455"
#> [1] "5 0.648768732557073"
#> [1] "6 0.267685335595161"
#> [1] "7 0.538673399714753"
#> [1] "8 0.239854503422976"
#> [1] "9 0.13323193253018"
```

For solving with a loop this basic knowledge can help to increase the pace of the algorithm.

``` r

rm(list=ls())
gc()
#>          used (Mb) gc trigger (Mb) max used (Mb)
#> Ncells 555536 29.7     940480 50.3   750400 40.1
#> Vcells 950416  7.3   12932172 98.7 11083794 84.6

row=100
col=100

A<<-matrix(runif(row*col),nrow = row,ncol = col)
B<<-matrix(runif(row*col),nrow = row,ncol = col)

CR<-matrix(A,nrow = row^2,ncol = col^2)
CC<-matrix(A,nrow = row^2,ncol = col^2)


### By row 
system.time({
  for (i in 1:row^2){
    IB=(i-1)%%row+1
    IA=floor((i-1)/row+1)
      for(j in 1:col^2){
        JB=(j-1)%%col+1
        JA=floor((j-1)/col+1)
        CR[i,j]<-A[IA,JA]*B[IB,JB]
      }
    }
})['elapsed']
#> elapsed 
#>   32.09



### By column 
system.time({
  for(j in 1:col^2){
    JB=(j-1)%%col+1
    JA=floor((j-1)/col+1)
    for (i in 1:row^2){
      IB=(i-1)%%row+1
      IA=floor((i-1)/row+1)
      CC[i,j]<-A[IA,JA]*B[IB,JB]
    }
  }
})['elapsed']
#> elapsed 
#>   31.72


sum(CC-CR)
#> [1] 0
```

#### Fun way

There are several packages for parallelizing:

-   doParallel
-   Parallel
-   snow
-   multicore

Some of those packages are required by the others, so lets start with **doParallel**. Once we open this package in our session it will require **parallel** that will automatically open the rest.

The easiest way to parallelize the loop above is using the package **foreach** in conjunction with **doParallel**. For this it's necessary to call the *registerParallel* function.

``` r

library(doParallel)
#> Warning: package 'doParallel' was built under R version 3.4.4
#> Loading required package: foreach
#> Warning: package 'foreach' was built under R version 3.4.4
#> Loading required package: iterators
#> Warning: package 'iterators' was built under R version 3.4.4
#> Loading required package: parallel

(no_cores<-detectCores()-1)
#> [1] 7

registerDoParallel()

getDoParWorkers()
#> [1] 3
```

Now we proceed to parallelize:

``` r

system.time({
  C<-foreach(A2=A,.combine = 'cbind',.packages = 'foreach')%dopar%{
        foreach(i=1:length(A2),.combine = 'rbind')%do%{
            A2[i]*B
        }
  }
})['elapsed']
#> elapsed 
#>    3.81


sum(CC-C)
#> [1] 0
```

Anytime we initialize the cores using **registerDoParallel** there is no need to close the connections, they will be stopped automatically once the program detects they aren't used anymore.

### Life is always a tradeoff

You might be wondering why we are not using more than 3 cores. Let's do it!

First, we need to introduce some new functions:

-   **makeCluster**: initializes all the cores or clusters that we require.
-   **stopCluster**: stops and releases the cores that we have initialized.

``` r

Times<-rep(0,no_cores)

for(h in 1:no_cores){
  cls<-makeCluster(h)
  registerDoParallel(cls)
  Times[h]<-system.time({
      foreach(A2=A,.combine = 'cbind',.packages = 'foreach')%dopar%{
        foreach(i=1:length(A2),.combine = 'rbind')%do%{
            A2[i]*B
        }
  }
  })['elapsed']
  stopCluster(cls)
}
```

<img src="Github_files/figure-markdown_github/Time_graph.png" width="500" />

Given that we are sharing memory, i.e., in every iteration we are calling *B*, the way *R* in Windows handles this is making the different cores wait until the others finish using *B*. Therefore, more cores doesn't always mean less processing time.

Divide and Conquer
------------------

Now we will divide the task into subtasks that will be processed by different nodes (called workers) and that at the end will be managed by a master.

``` r


library(doParallel)

# Checking cores
parallel::detectCores()


doichunk<-function(chunk){
  require(doParallel)
  N=length(chunk)
  col=ncol(B)
  row=nrow(B)
  MatC<-foreach(A2=A[,chunk],.combine = 'cbind',.packages = 'foreach')%dopar%{
    foreach(i=1:length(A2),.combine = 'rbind')%do%{
        A2[i]*B
    }
  }
  return(MatC)
}


ChunkKronecker<-function(cls,A,B){
  clusterExport(cls,c('A','B'))
  ichunks<-clusterSplit(cls,1:ncol(A)) # Split jobs
  tots<-clusterApply(cls,ichunks,doichunk)
  return(Reduce(cbind,tots))
}



Tiempos<-rep(0,30)

for(i in 1:30){
  cls<-makeCluster(i)
  Tiempos[i]<-system.time(valor<-ChunkKronecker(cls,A,B))['elapsed']
  stopCluster(cls)
}

sum(CC-valor)
```

<img src="Github_files/figure-markdown_github/ChunksTimes.png" width="500" />

Submitting R scripts into the cluster
=====================================

From now on the operating system will be Linux. Linux and Windows have significant differences from each other, not only in the graphic user interface, but also in their architectural level. One of the main differences is that Windows doesn't fork and Linux does. That means that Linux make copies of the needed variables in all the cores, instead of forcing the cores to queue and wait for using the shared variables.

The basics
----------

First, open *R* in the terminal, install the packages and close *R*. When the packages are installed for the first time, it takes about 10 minutes to download everything.

``` r

> R

install.packages("Rmpi", dependencies=TRUE, configure.args=c("--with-mpi=/cm/shared/apps/openmpi/gcc/64/1.10.7"))
install.packages("doParallel", dependencies=TRUE) ## CRAN Mirror 34 (Germany)


library(doParallel)

(no_cores<-detectCores()-1)

registerDoParallel()

getDoParWorkers()

q()
```

Lets check what happens if instead of **makeCluster** we use **makeForkCluster** (attention, this function is only available for Linux) for the Kronecker product.

``` r


library(doParallel)

(no_cores<-detectCores()-1)


#### Normal ####

TimesN<-rep(0,no_cores)

for(h in 1:no_cores){
  cls<-makeForkCluster(h)
  registerDoParallel(cls)
  TimesN[h]<-system.time({
      foreach(A2=A,.combine = 'cbind',.packages = 'foreach')%dopar%{
        foreach(i=1:length(A2),.combine = 'rbind')%do%{
            A2[i]*B
        }
  }
  })['elapsed']
  stopCluster(cls)
}


#### Master - Workers ####

doichunk<-function(chunk){
  require(doParallel)
  N=length(chunk)
  col=ncol(B)
  row=nrow(B)
  MatC<-foreach(A2=A[,chunk],.combine = 'cbind',.packages = 'foreach')%dopar%{
    foreach(i=1:length(A2),.combine = 'rbind')%do%{
        A2[i]*B
    }
  }
  return(MatC)
}


ChunkKronecker<-function(cls,A,B){
  clusterExport(cls,c('A','B'))
  ichunks<-clusterSplit(cls,1:ncol(A)) # Split jobs
  tots<-clusterApply(cls,ichunks,doichunk)
  return(Reduce(cbind,tots))
}



TimesC<-rep(0,no_cores)

for(i in 1:no_cores){
  cls<-makeForkCluster(i)
  TimesC[i]<-system.time(valor<-ChunkKronecker(cls,A,B))['elapsed']
  stopCluster(cls)
}


Times<-data.frame(NoCores=1:no_cores,Normal=TimesN, Workers=TimesC)

write.csv(Times,'TimesFork.csv')
```

For running and R code in terminal, without opening *R*, just write **Rscript** followed by the name of your *R* file.

``` r

> Rscript KroneckerLinux.R
```

How does the cluster work?
--------------------------

For submitting jobs to the cluster it is necessary to create a **shell file** using a **batch system** through LSF commands. So, the GWDG cluster is operated by the LSF platform, which is operated by shell commands on the frontends. The **frontends** are special nodes (gwdu101, gwudu102, and gwdu103) provided to interact with the cluster via shell commands.

The batch system distributes the processes across job slots, and matches the job's requirements to the capabilities of the job slots. Once sufficient suitable job slots are found, the job is started. LSF considers jobs to be started in the order of their priority.

It is also necessary to know how the cluster is structured:

<img src="Github_files/figure-markdown_github/GWDGarchitecture.PNG" alt="source: https://info.gwdg.de/dokuwiki/doku.php?id=en:services:application_services:high_performance_computing:running_jobs" width="100%" />
<p class="caption">
source: <https://info.gwdg.de/dokuwiki/doku.php?id=en:services:application_services:high_performance_computing:running_jobs>
</p>

### Basic options of **disk space**

-   /scratch: this is the shared scratch space, available on *gwda, gwdc*, and *gwdd* nodes and on the frontends *gwdu101* and *gwdu102*. For being sure of having a node with access to shared */scratch* the command **-R scratch** must be written in the shell file.

-   /scratch2: this is the shared scratch space, available on *dfa, dsu, dge*, and *dmp* nodes, and on the frontend *gwdu103*. For being sure of having a node with access to shared */scratch2* the command **-R scratch2** must be written in the shell file.

-   $HOME: our home directory is available everywhere, permanent, and comes with backup, but it is comparatively slow.

### Basic queue *mpi*

**mpi** is the general purpose queue, usable for serial and SMP jobs with up to 20 tasks, but it is especially well suited for large MPI jobs. Up to 1024 cores can be used in a single MPI job, and the maximum is 48 hours. This is specified in the shell file through the command **-q mpi**.

Some extra parameters for this queue are:

-   -long: the maximum run time is increased to 120 hours. Job slot availability is limited, and long waiting times are expected.

-   -short: the maximum run time is decreased to two hours. In turn the queue has a higher base priority, but it also has limited job slot availability. That means that as long as only few jobs are submitted to the *-short* queues they will have minimal waiting times.

### Specifying node properties with *-R*

**-R** runs the job on a host that meets the specified resource requirements. Some of its basic parameters are:

-   span\[hosts=1\]: this puts all processes on one host.
-   span\[ptile=&lt; x &gt;\]: *x* denotes the exact number of job slots to be used on each host. If the total process number is not divisible by *x*.
-   scratch(2): the node must have access to *scratch(2)*.

### Last but not less important parameters

-   -N: sends the job report to you by email when the job finishes.
-   -u: sends the email to the specified email destination.
-   -W: sets the runtime limit of the job.
-   -o: appends the standard output of the job to the specified file path.
-   -n: submits a parallel job and specifies the number of tasks in the job.
-   -a: this option denotes a wrapper script required to run SMP or MPI jobs. The most important wrappers are[1]:
    -   Shared Memory (**openmp**). This is a type of parallel job that runs multiple threads or processes on a single multi-core machine. OpenMP programs are a type of shared memory parallel program.
    -   Distributed Memory (**intelmpi**). This type of parallel job runs multiple processes over multiple processors with communication between them. This can be on a single machine but is typically thought of as going across multiple machines. There are several methods of achieving this via a message passing protocol but the most common, by far, is MPI (Message Passing Interface).
    -   Hybrid Shared/Distributed Memory (**openmpi**). This type of parallel job uses distributed memory parallelism across compute nodes, and shared memory parallelism within each compute node. There are several methods for achieving this but the most common is OpenMP/MPI.

### The **bsub** command: Submitting jobs to the cluster

**bsub** submits information regarding your job to the batch system. Instead of writing a large *bsub* command in the terminal, we will create a shell file specifying all the *bsub* requirements.

The shell files can be created through either the linux terminal using the editor *nano* or Notepad++ in Windows.

Non-parallel jobs
-----------------

For creating the shell file we will use the Linux text editor **nano**.

``` r

> nano NameShell.sh

------------------------
#!/bin/sh 
#BSUB -N 
#BSUB -u <YourEmail>
#BSUB -q mpi 
#BSUB -W <max runtime in hh:mm>
#BSUB -o NameOutput.%J.txt 
 
R CMD BATCH NameCode.R
--------------------------
```

For submiting the job the command is:

``` r
> bsub < NameShell.sh
```

Parallel jobs
-------------

``` r

> nano NameShell.sh

------------------------
#!/bin/sh 
#BSUB -N 
#BSUB -u <YourEmail>
#BSUB -q mpi 
#BSUB -W <max runtime in hh:mm>
#BSUB -o NameOutput.%J.txt 
#BSUB -n <min>,<max> or <exact number> 
#BSUB -a openmp 
#BSUB -R span[hosts=1] 
 
R CMD BATCH NameCode.R
--------------------------
```

``` r
> bsub < NameShell.sh
```

So, for submiting our job the shell file would look like the next one:

``` r

> nano KroneckerLinux.sh

------------------------
#!/bin/sh 
#BSUB -N 
#BSUB -u gil@demogr.mpg.de
#BSUB -q mpi 
#BSUB -W 00:30
#BSUB -n 8 
#BSUB -a openmp 
#BSUB -R span[hosts=1] 
 
R CMD BATCH KroneckerLinux.R
--------------------------
```

``` r
> bsub < KroneckerLinux.sh
```

<img src="Github_files/figure-markdown_github/SUBMIT.PNG" width="800" />

<img src="Github_files/figure-markdown_github/email.PNG" width="500" />

<img src="Github_files/figure-markdown_github/ForkVSnoFork.png" width="500" />

Other important LSF commands
----------------------------

-   bjobs: lists currents jobs.

While not having a job slot:

<img src="Github_files/figure-markdown_github/WAITING.PNG" width="800" />

Once having a job slot:

<img src="Github_files/figure-markdown_github/WORKING.PNG" width="800" />

-   bhist: lists older jobs.
-   lsload: status of cluster nodes.
-   bqueues: status of cluster nodes.
-   bhpart: shows current user priorities.
-   bkill <jobid>: stops the current job.

References
==========

<img src="Github_files/figure-markdown_github/Book.jpg" width="200" />

<https://info.gwdg.de/dokuwiki/lib/exe/fetch.php?media=en:services:scientific_compute_cluster:parallelkurs.pdf>

[1] <https://wiki.uiowa.edu/display/hpcdocs/Advanced+Job+Submission>
