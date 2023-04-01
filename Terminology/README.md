# Common Terminology

> ### Toil

- Manual
- Repetitive
- Automatable

> ### Overhead

Overhead is often work not directly tied to running a production service, and includes tasks like team meetings, setting and grading goals,19 snippets,20 and HR paperwork.

> ### Service Level Indicators

Carefully define quantitative measurement of a service/system generally of keys metrics, usually via observability tools.

> ### Service Level Objectives

Once a system has been measured certain benchmarks for target ranges can be determined for a service level given the SLIs.

> ### Service Level Agreements

An explicit/implicit contract with a systems users that includes consequences of meeting (or missing) the SLO they contain.

> ## Four Golden Signals

<b>Traffic</b> <br>
A measurement of how much demand is being placed on a system, measured in a high-level system-specific metric. Usually denoted for APIs as request per second (RPS).

<b>Latency</b> <br>
The time it takes to service a request. Difference between a successful request HTTP 200 vs non successful request latencies !HTTP 200 (400s, 500s).

<b>Saturation</b> <br>
How much load is on your system. A measure of your systems fraction, emphasizing the resources that are most constrained (e.g. in a memory constrained system, show memory, etc...).

<b>Errors</b> <br>
The rate of which requests fails, either explicitly (HTTP 500), implicitly (an HTTP 200, but with bad content). Generally total availability can be determined from 1/error rate.

> ### Objectives and Key Results (OKRs)
<b>Objectives</b></br>
Simply is what is to be achieved. Significant, concrete, and action oriented. "What you want to accomplish"

<b>Key Results</b></br>
The yard stick of how to measure progress to the objectives. Specific,time-bound, aggressive yet realistic. "3-5 milestones of how you will accomplish it". 



