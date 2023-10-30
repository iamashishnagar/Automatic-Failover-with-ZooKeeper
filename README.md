# Automatic-Failover-with-ZooKeeper
Zookeeper-supported Automatic Failover in a Master-Worker Distributed Execution

We need jar files from

1. apache-zookeeper-3.7.1-bin/lib
2. https://www.apache.org/dyn/closer.lua/logging/log4j/2.18.0/apache-log4j-2.18.0-bin.tar.gz


This project uses Zookeeper to implement an automatic failover mechanism in a master-worker distributed execution of graph-bridge programs. Client.java (i.e., the master) connects to ZooKeeper;
submits 10 different tasks, each executing GraphBride.java with a different graph size, and waits for all tasks to be completed by remote worker processes Each worker should be implemented in Worker.java joins Zookeeper; repetitively starts a new task; and fails over a task if it is not done within 100 seconds. Distributed synchronization must be implemented in Key.java to have all the workers access the bag of tasks exclusively.

**Client.java** Connects to ZooKeeper at a given TCP port in args[0], creates the /workers and the /tasks nodes persistently; submits 10 tasks under /tasks where each task has “submitted” as its data and is identified with task-000000000d where d = 1, 2, …, 10; launches 10 event watchers, each checking a task deletion from /tasks; and finally deletes /workers and /tasks from ZooKeeper upon no more tasks under /tasks.

**Worker.java** Joins ZooKeeper at a given TCP port in args[0]; registers itself as worker- 00000000d where d = 1, 2, …, 10 under /workers; and repeats picking up a new task until /tasks become empty. For each task picked up, Worker.java checks its data: “submitted” allows the worker to update its data with the current timestamp and to launch a new GraphBridge program, otherwise the data should be a past timestamp when someone else picked up this task. In that case, Worker.java checks if the task is overdue beyond 100,000msec. If so, let’s restart this task. Otherwise, Worker.java simply leaves the current task execution, assuming that the task may be stalled.


**Key.java** Implements lock( ) and unlock( ). They are used for each worker to exclusively access the /tasks node when picking up and updating a task- 000000000d in a non-interruptive fashion. The lock( ) method creates the /lock node. If the node has been already created, (i.e., someone else has locked), the worker launches an event watcher and waits on itself. The event watcher is woken up upon deletion of /lock and notifies the worker. The unlock( ) method simply deletes the /lock node.

**GraphBridge.java** Is a Java application to be executed by Worker.java. It receives the number of vertices, (say N) in args[0]; generates a random graph with N vertices; and finds all graph bridges using depth-first search. You don’t have to understand the details of the algorithm.

**Worker.java**
The Worker program efficiently processes tasks by continuously selecting and executing available tasks until completion. It employs the pickupTask(), runTask(), and finishTask() methods for task handling. In pickupTask(), tasks are retrieved and assessed based on their status and timestamps. Tasks marked as "submitted" or those with timestamps exceeding 100 seconds are selected for execution. If all remaining tasks are being executed within the time limit, it returns "job stalled." The runTask() method performs the task-specific computation by converting the taskID to determine the number of vertices and executing a corresponding command. The task's output is displayed on the standard output. Upon task completion, the finishTask() method updates the task's status and removes it from the task list. By seamlessly integrating these methods, the Worker program efficiently manages task processing, ensuring a continuous and smooth execution flow until all tasks are finished.

**Key.java**
The Key program facilitates the acquisition and release of locks for Workers to ensure uninterrupted access to the "/tasks" znode in a distributed system. Constructed with a Worker's ZooKeeper object, the Key sets up a synchronization object. The lock() method attempts to acquire the lock on the "/lock" znode. If successful, the Worker gains access to the tasks. If another Worker holds the lock, the method sets up a lockWatcher and waits on the synchronization object. When the "/lock" znode is deleted, the lockWatcher notifies the waiting Worker to resume execution. The unlock() method releases the lock by deleting the "/lock" znode. By utilizing the Key program, Workers can safely interact with and manipulate tasks in the "/tasks" znode, ensuring smooth and interference-free task processing in a distributed environment.

**Additional feature - Run any number of tasks rather than only 10 tasks (Client.java)**
The Client program serves as the coordinator for task management in a distributed system. It establishes and maintains two essential znodes: "/workers" and "/tasks". The number of tasks submitted is determined by the nTasksSubmitted variable, which defaults to 10 if not specified via the command line. The program initiates by joining the ZooKeeper session and waiting for connection establishment. Once connected, the Client creates the "/workers" znode to indicate the presence of workers in the system. It proceeds to create the "/tasks" znode and submits a series of tasks named "/tasks/task-00000000d" (where d ranges from 00 to nTasksSubmitted-1), with each task marked as "submitted". To ensure task completion, the Client launches a watcher for each task and verifies the deletion of all tasks. It deletes the "/tasks" znode once all tasks are processed. Subsequently, the program monitors the absence of workers by observing the "/workers" znode. Finally, the Client removes the "/workers" znode. The significance of the nTasksSubmitted variable lies in determining the number of tasks submitted by the Client, allowing flexibility in the number of tasks based on the provided command-line argument. Overall, the Client program plays a pivotal role in initiating and coordinating task submission and monitoring in a distributed environment.
