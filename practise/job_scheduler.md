# Problem
What is a Job Scheduler
A job scheduler is a program that automatically schedules and executes jobs at specified
times or intervals. It is used to automate repetitive tasks, run scheduled maintenance, or
execute batch processes. There are two key terms worth defining before we jump into solving the problem:
Task: A task is the abstract concept of work to be done. For example, "send an email".
Tasks are reusable and can be executed multiple times by different jobs.
Job: A job is an instance of a task. It is made up of the task to be executed, the schedule for
when the task should be executed, and parameters needed to execute the task. 
The main responsibility of a job scheduler is to take a set of jobs and execute them according
to the schedule.

# Solution Approach

## Assumptions
1. FaaS: A task is defined as a collection of (function, arguments, runtime, schedule). When the scheduled time comes, the service starts the runtime - executes the functions against the arguments. 
2. The arguments take care of fetching and reading the data; the service is responsible for storing the intermediate log files that the function generates as well as the status of the function call (ok, running, not ok etc)
3. User should be able to onboard new function, modify existing function, and deprecate task
---decided this as I made the design----
4. Access control is out of scope -- assume that user modifying task has permissions
5. Geography is out of scope -- we assume jobs can run anywhere


## Approximations
1. Running the jobs - N jobs per day, each job creates 1 Log file (size X / file) and updates the status table 
2. T tasks - Alpha modifications per day 

## Clarifications
1. How is the function and run-time stored? For now, let us assume function is stored as blob along with arguments

## Constraints
1. Jobs must start immediately after being scheduled (if I schedule a task for 1AM at 12:59AM, it should pick it up?)
2. Jobs cannot run indefinitely -- cut of after some pre-scheduled time (configuration in runtime/ system max cut off) and 
   status should be updated as required
3. 

## Incentives / Trade-offs
1. Minimize compute cost of FaaS
2. Maximize on-time starting/ completion of jobs

## Decisions/ Design 
Think about this as a Discrete Event Simulation -- there are two components -- a Scheduler that "schedules" jobs onto a priority queue; and an Executor that executes the tasks. in  a DES, we maintain a priority queue because the simulation can jump time arbitrarily. Over here, it feels like the schedule needs to be something like a cache of {run_time: [job_ids]} which the executor then pulls thee job_ids at the run_time and starts running them.

Scheduler - I think we have two options -- first one is that when the task is created, we populate all the jobs (upto a reasonable end date) and write it to a DB; and then the online scheduler just chunks the next N hours of job from the DB. Any new job that needs to be run in the next N hours - gets automatically added both to the DB and the online scheduler.  When the window refreshes for the online scheduler, jobs that are already in the online scheduler are not re-scheduled (based on the job id). In this system, when a task is submitted, we create DB entries (Year, month, day, hour, minute, second, task_id, job_id, is_active=True). The Executor actually reads the task_id to find out the "latest" parameters to execute the task. 

The second option is that the scheduler "creates" the schedule by reading the entire Task DB (more likely the subset that is still active. Over here, it makes the computation of which Tasks from the task_DB need to fire in the next N minutes (that are not already loaded into the scheduler). For each task_id is creates the same (run_time: (task_id, job_id)} data and keeps it in memory. 

I prefer the first option here where users modifying jobs are decoupled (except for the next N minute window of active jobs in the scheduler, where we have to modify the existing jobs in the scheduler)

Orchestrator - This module "prepares" the jobs for the executor. It looks for the next N jobs from the scheduler, and then pulls the relevant functions, runtimes and parameters from the blob storage based on the task_id. Note that if the user has changed the inputs etc for the same task_id, this would pick it up. Here, we can use historical data from our observeability metrics to also pick the right VM size to run the job in etc.

Executor - It picks up the job from the orchestrator and diverts the job to the right VM that was identified. It manages a pool of VMs where the jobs are executed, keeping track of the jobs that are already running etc to make a decision on potentially reusing an existing VM or provisioning a new one. On starting a task, it updates ours TaskDB with start_Time with status running. It monitors jobs and kills them if they run longer than the scheduled run_time -- as well as collects metrics + logs from the VM to emit to our TaskDB and MetricsDB>. My mental model for the executor is like a thread pool manager. 

Failure Modes
Scheduler goes down -- Keep active and backup schedulers running

Too many jobs -- Have multiple schedulers and executors running


# After Reading Hello Interview
- Design was mostly correct -- I had more description of the Scheduler/ Executor etc where the Hello Interview folks started talking about existing technologies (Redis sorted set for the scheduler, EC2 Container service for the Executor etc)

- Constraints and Assumptions - Looked mostly okay based on the Hello Interview answer. 

- Identified the "hot path" -- where a new job lands within the execution window

-  Needs improvement -- Did not discuss Partition / Primary Key etc of my Tables or the schema etc.Kept going back to the table to add thing whereas the Hello Interview folks seem to suggest the schema beforehand (but they also have the benefit of hindsight - I do not know what is the right solution here)

- Global Secendoary Index for the execution status -- I think I just wrote it to a table with all the indices

Overall, I think I did this problem well -- but I had background knowledge -- been involved/ thinking about such systems w/ my current job. 
