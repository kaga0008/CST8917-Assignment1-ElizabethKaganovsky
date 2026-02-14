# CST8917 Assignment 1: Serverless Computing - Critical Analysis
## Elizabeth Kaganovsky (040956095) - February 13, 2026

### Part 1. Paper Summary
Hellerstein et al.'s paper _Serverless Computing: One Step Forward, Two Steps Back_ explores the applications of then-nascent  functions, a cloud-offered service that bills itself as a simple serverless solution for a variety of use cases with a more flexible billing model than the typical always-on server. Functions in a cloud context are intended to mirror functions in the traditional programming zeitgeist--mappings from inputs to outputs as carried out by small bits of code--but with the added advantage of simplified deployment and autoscaling, enabling developers to avoid the burden of managing their scaling with hand-made scripts. 

While appealing on a conceptual level, Hellerstein et al. identify significant drawbacks to the serverless paradigm and major weaknesses in functions as a service (FaaS). The first of multiple issues is that of execution times--while a function may reap the benefits of being a miniscule amount of code, the process of starting up a function without a preloaded "warm" VM is often orders of magnitude slower than getting a response from a function that was recently executed and persists in memory until its time to live expires. From this stems the issue of I/O bottlenecks--when multiple functions are packed onto a warm VM to save on setup times, they paradoxically may cumulatively slow down as each invocation shares the same limited bandwidth allocated to that VM. 

Additionally, the process of passing persisting data between functions or other cloud resoures involves time-intensive operations on storage resources due to the lack of direct addressibility between instances of a function, which further drag out the execution time and requires an intermediary service such as a storage account, necessitating expensive reads and writes between the two. The necessity of peripheral resources make fuctions a less "compact" solution than advertised.

Due to the isolated natures of function, executing on VMs without direct access to the data resources they interact with. This is remedied by simply sending the data to the VM that the function is executing on--but in doing this falls prey to the "data-to-code" anti-pattern where network bandwidth is throttled by streams of data being "shipped to the code." The reverse "code-to-data" pattern involves shipping the code to the same machine the relevant data is stored, massively saving on bandwith usage due to the miniscule size of the function, though this pattern is curiously not utilized by major CSPs, at least at the time of publication.

Functions also lack access to customizable hardware, notably GPUs, which limit their usability in fields such as machine learning, and lack support for main-memory databases that keep data in RAM, further enforcing the above issue with accessing data efficiently. These combined factors, the authors argue, therefore make serverless functions a poor form of utilizing the distributed computing environment of the cloud.

As guidance, the authors offer suggestions for improving serverless functions by addressing their aforementioned concerns. Most clearly beneficial is to establish some method of persisting resources with known identities between application elements with affinities for one another (though not _explicitly_ defined, I interpret "affinities" in this context to refer to repeated connections between certain elements, i.e. a function accessing a specific database on every invocation, or running on a certain hardware element repeatedly), which can scale and remap across physical resources rapidly and efficiently to accomodate for the virtualized environment. Additionally, methods of colocating data and code would be beneficial--while issues will innevitably arise due to the medium of virtualized software, reducing the amount of networked connections between resources and instead "bringing the code to data" will reduce overall latency significantly. Addressing the limitations on hardware would further open up serveless functions to additional use cases, particularly those which require hardware acceleration or GPUs.[1]

### Part 2. Azure Durable Functions Deep-Dive
#### 2.2. Orchestration model
***How do orchestrator, activity, and client functions work together? How does this differ from basic FaaS?

The orchestration model is an architecture provided by Azure Durable Functions (ADF) which partitions functions into three types--orchestrators, activity, and client. The process of an application's execution begins with a client function, the entry point into a larger application. Any non-orchestrator function can be a client function. [2] An orchestrator function contains logic for calling additional functions, as well as managing the state of a workflow. Activity functions are those which perform individual tasks (single units of work) and can be executed in series, in parallel, or in any combination of the two, and allow for much longer running, complex architectures than those allowed by basic functions.

In comparison, basic functions lack this diversification and can only execute in series. The lack of types of functions mean that every basic function is essentially an activity function performing small units of work one after the other, communicating through reads and writes to shared data resources.

#### 2.3. State management
***How does Durable Functions manage state (event sourcing, checkpointing, replay)? How does this address the paper's criticism that functions are stateless?

Azure Durable Functions manage state through the use of persistent storage. Starting an ADF creates an append-only storage table named (by default) DurableFunctionsHubHistory that records the execution state and events throughout the lifetime of an ADF workflow. The use of a storage table recording events is also the method through which checkpointing is achieved--recording state at every await()/yield() call such that in the event of a failure, the orchestrator can retry from the state of the last successful function execution[3]. Here is where idempotency is important; as activity functions have an "at least once" execution guarantee, it is possible for any individual activity function to execute multiple times, which can cause cascading errors when not designed with multiple executions in mind[4] (for example: In a workflow processing a user's order on a website, a network issue causes the activity function that handles charging the user's card executes several times as the orchestrator has to retry from a checkpoint--without logic to accomodate this, the user will be charged for each execution).

This method addresses Hellerstein et al.'s criticisms regarding statelessness relying on expensive reads and writes to an intermediary storage account. By using an inexpensive append-only table, expensive data access operations can be saved for necessary cases rather than relying on them as a shoddy replacement for more effective state management.

#### 2.4. Execution timeouts
*** How do orchestrators bypass the timeout limits that apply to regular Azure Functions? What limits still apply to activity functions?

The use of stateful storage by ADF is the method through which orchestrators bypass the time limits applied to typical functions. Orchestrators can use a durable timer to checkpoint its state to persistent storage and then terminate. On reset of the timer, the function is called again and resumes where it left off based on the events recorded in the previous execution [5]. Activity functions, however, are still subject to the typical expiration limits depending on their consumption plan.

#### 2.5. Communication between functions
*** How do orchestrator and activity functions communicate? Does this address the paper's criticism about functions needing slow storage intermediaries?

ADF notably do not solve the lack of direct network addressability between functions. In place of direct communication, ADF use task hubs. However, they improve upon the communication model of typical functions through use of a Task Hub, a queue-like data store that manages the table of execution history and checkpointing, as well as messages being passed between functions. Depending on the storage provider, task hubs can be represented in storage differently, with options available depending on the needs of the application [6].

#### 2.6. Parallel execution (fan-out/fan-in)
*** How does the fan-out/fan-in pattern work? How does it address the paper's concern about distributed computing?

Fan-out/fan-in in an architectural pattern that can be used in a ADF for the parallel execution of functions. Multiple function instances can be triggered at the same time by an orchestrator function, diverging to accomplish each unit of work independently ("fan-out"). The orchestrator waits on results to return from each activity function ("fan-in") before continuing

### Part 3. Critical Evaluation
#### 3.1. Limitations that remain unresolved
#### 3.2. Your Verdict

### Part 4. References
[1] original paper go here
[2] https://learn.microsoft.com/en-us/azure/azure-functions/durable/durable-functions-types-features-overview
[3] https://learn.microsoft.com/en-us/azure/azure-functions/durable/durable-functions-orchestrations?tabs=csharp-inproc
[4] https://www.linkedin.com/learning/learning-azure-durable-functions/introduction-to-durable-functions?u=2199673
[5] https://learn.microsoft.com/en-us/azure/azure-functions/durable/durable-functions-timers?tabs=csharp
[6] https://learn.microsoft.com/en-us/azure/azure-functions/durable/durable-functions-task-hubs?tabs=csharp

### part 5. AI Disclosure
No AI tools were used in this assignment, except for a few times when I Googled something and the AI overview included a link to an actual source.