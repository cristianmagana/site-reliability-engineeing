# Common Terminology

> ### Toil
<br>

Toil is the kind of work tied to running a production service that tends to be manual, repetitive, automatable, tactical, devoid of enduring value, and scales linearly as the service grows. According to Google's SRE principles, teams should aim to keep toil below 50% of their time to maintain engineering focus and prevent burnout.

<br>

<b>Characteristics of Toil</b> <br>

**Manual**: Work that requires human intervention. Even if running a script is part of the task, if you have to manually trigger it each time, it's toil.

**Repetitive**: Work that you perform over and over again. If you're solving the same problem repeatedly rather than fixing it once, it's toil.

**Automatable**: Work that could be automated. If a machine could do the task just as well or better, it's toil.

**Tactical**: Interrupt-driven, reactive work rather than proactive engineering. Firefighting production issues rather than preventing them.

**No Enduring Value**: Work that doesn't permanently improve the service. The service remains in the same state after the work is completed.

**Scales with Service Growth**: Work that grows linearly (or worse) with service traffic, size, or complexity. If your service doubles in size and your work doubles too, that's toil.

<br>

<b>Examples of Toil</b> <br>

- **Manual deployments**: SSH into servers and manually run deployment commands for each release
- **Ticket-driven operations**: Manually provisioning user accounts, resetting passwords, granting access one request at a time
- **Log diving**: Manually searching through logs to diagnose issues rather than having automated alerting and debugging tools
- **Capacity management**: Manually adding servers or storage when utilization reaches thresholds
- **Data pipeline babysitting**: Manually restarting failed batch jobs or data processing tasks
- **Configuration updates**: Manually updating config files across multiple servers for routine changes

<br>

<b>Impact and Mitigation</b> <br>

**Negative impacts of excessive toil**:
- Engineering team burnout and low morale
- Reduced velocity on project work and innovation
- Increased operational costs as systems scale
- Higher error rates due to manual intervention
- Difficulty attracting and retaining talent

**Strategies to reduce toil**:
- **Automation**: Build tools and scripts to eliminate repetitive manual work
- **Self-service platforms**: Enable users to perform common tasks themselves (password resets, provisioning)
- **Infrastructure as Code (IaC)**: Manage infrastructure through version-controlled code
- **CI/CD pipelines**: Automate testing, building, and deployment processes
- **Improved monitoring and alerting**: Detect and diagnose issues automatically
- **Error budgets**: Use error budgets to prioritize reliability work vs feature work

Google's SRE guideline: If your team spends more than 50% of time on toil, it's time to invest in automation and tooling to reclaim engineering capacity.

<br> <br> 
> ### Overhead
<br>

Overhead refers to administrative and organizational work that is necessary for running a team but not directly tied to running production services or delivering features. Unlike toil (which is production-related but should be automated), overhead is essential organizational work that typically cannot be automated away.

<br>

<b>Common Types of Overhead</b> <br>

**Meetings and Communication**:
- Team stand-ups, sprint planning, and retrospectives
- Cross-functional alignment meetings
- All-hands and organizational updates
- 1:1s with managers and skip-level meetings

**Planning and Goal Setting**:
- OKR planning and review sessions
- Performance review cycles (self-assessments, peer reviews)
- Project planning and roadmap discussions
- Capacity planning and resource allocation

**Administrative Tasks**:
- HR paperwork and compliance training
- Expense reports and time tracking
- Equipment requests and access provisioning
- Documentation of team processes and runbooks

**Hiring and Onboarding**:
- Interview candidate screening and interviews
- Writing job descriptions
- Onboarding new team members
- Mentoring and training

<br>

<b>Managing Overhead</b> <br>

**Healthy overhead** (typically 20-30% of time):
- Enables effective team coordination
- Ensures alignment with organizational goals
- Builds team culture and knowledge sharing
- Necessary for career development

**Excessive overhead** (>40% of time) becomes problematic:
- Reduces time for actual engineering work
- Causes meeting fatigue and context switching
- Lowers team productivity and morale
- Signals organizational inefficiency

**Strategies to optimize overhead**:
- **Meeting hygiene**: Clear agendas, time limits, required vs optional attendance
- **Async communication**: Use documentation and written updates when possible
- **Batching**: Consolidate similar overhead activities (all interviews on certain days)
- **Delegation**: Distribute overhead fairly across the team
- **Regular audits**: Periodically review and eliminate low-value overhead

The key is balancing necessary coordination and planning (overhead) with engineering work and automation efforts (reducing toil).

<br> <br>

> ### Service Level Indicators (SLIs)
<br>

Service Level Indicators are carefully selected quantitative measurements that directly reflect the user experience of a service. SLIs form the foundation of reliability engineering by providing objective data about how well a system is performing from the user's perspective.

<br>

<b>Characteristics of Good SLIs</b> <br>

**User-centric**: Measure what users actually care about, not just what's easy to measure
**Quantifiable**: Must be objectively measurable with numbers (not subjective opinions)
**Actionable**: Should inform decisions about when to improve reliability vs ship features
**Focused**: Typically 2-5 key metrics that matter most; too many dilutes focus

<br>

<b>Common SLI Types</b> <br>

**Availability/Success Rate**: Proportion of successful requests vs total requests
- Example: 99.9% of API requests return HTTP 200 (not 5xx errors)
- Formula: (successful requests / total requests) × 100

**Latency**: Time taken to serve a request
- Example: 95% of requests complete in under 200ms
- Measured at various percentiles (p50, p95, p99) to catch outliers

**Throughput**: Volume of requests processed successfully
- Example: System handles 10,000 requests per second
- Important for capacity planning and scaling

**Correctness/Quality**: Accuracy of the output
- Example: 99.99% of data processing jobs produce correct results
- Often measured by comparing outputs to expected results

**Durability**: Data retention and persistence
- Example: 99.999999999% (11 nines) of stored data is retrievable
- Critical for storage systems and databases

<br>

<b>Real-World Examples</b> <br>

**E-commerce website**:
- Availability: 99.95% of page loads succeed
- Latency: p99 page load time < 2 seconds
- Conversion: 99% of checkout transactions complete successfully

**Video streaming service**:
- Availability: 99.9% of video streams start successfully
- Quality: 95% of playback time is at requested quality (no buffering)
- Latency: Video playback starts within 3 seconds

**Database service**:
- Availability: 99.99% of queries return successfully
- Latency: p95 query response time < 50ms
- Durability: 99.999999999% of writes are persisted

<br>

<b>Measuring SLIs</b> <br>

SLIs are typically collected through:
- **Application Performance Monitoring (APM)** tools: DataDog, New Relic, Dynatrace
- **Logging and metrics platforms**: Prometheus, Grafana, CloudWatch, Splunk
- **Synthetic monitoring**: Automated tests that simulate user behavior
- **Real User Monitoring (RUM)**: Actual user experience metrics from client-side

**Key principle**: Measure SLIs from the user's perspective, not just from internal systems. A server might be "up" but if users can't access it due to network issues, the SLI should reflect that failure.

<br><br>

> ### Service Level Objectives (SLOs)
<br>

Service Level Objectives are target values or ranges for SLIs that define the desired reliability of a service. SLOs represent a commitment between service providers and users about the expected quality of service. They answer the question: "How reliable should this service be?"

<br>

<b>Key Principles of SLOs</b> <br>

**Aspirational but achievable**: Set high enough to satisfy users but realistic given current capabilities
**Measurable**: Based on objective SLI metrics, not subjective feelings
**Business-aligned**: Should reflect business priorities and user expectations
**Error budget-driven**: The gap between 100% and your SLO (e.g., 99.9% SLO = 0.1% error budget)

<br>

<b>Setting SLOs</b> <br>

**Too strict** (e.g., 99.999% when 99.9% suffices):
- Unnecessary engineering costs
- Slows down feature development
- May prevent useful experimentation

**Too loose** (e.g., 95% when users expect 99.9%):
- Users experience poor service quality
- Loss of trust and customer churn
- Competitive disadvantage

**Finding the right balance**:
1. Understand what users actually need (not want)
2. Analyze historical performance data
3. Consider business impact of downtime
4. Account for dependencies on third-party services
5. Start conservative, iterate based on data

<br>

<b>Real-World SLO Examples</b> <br>

**API Service**:
- **Availability SLO**: 99.95% of requests succeed (allowing 0.05% error budget = ~21 minutes downtime per month)
- **Latency SLO**: 99% of requests complete in <100ms, 99.9% in <500ms
- **Throughput SLO**: Handle minimum 50,000 requests/second during peak hours

**Mobile App Backend**:
- **Availability SLO**: 99.9% of API calls return successfully (allowing ~43 minutes downtime per month)
- **Latency SLO**: p95 response time <200ms for critical endpoints
- **Data freshness SLO**: User data updates reflected within 5 seconds 99.9% of the time

**Data Pipeline**:
- **Completeness SLO**: 99.99% of data records processed successfully
- **Freshness SLO**: 99% of data available within 1 hour of generation
- **Accuracy SLO**: 99.999% of processed data matches source data

<br>

<b>Error Budgets</b> <br>

The error budget is the allowed failure rate implied by your SLO:
- **SLO of 99.9%** = **0.1% error budget** = ~43 minutes of downtime per month (30-day month)
- **SLO of 99.95%** = **0.05% error budget** = ~21 minutes of downtime per month
- **SLO of 99.99%** = **0.01% error budget** = ~4 minutes of downtime per month

**How to use error budgets**:
- **Budget remaining**: Continue shipping features quickly, take calculated risks
- **Budget exhausted**: Focus on reliability, freeze risky changes, pay down technical debt
- **Budget surplus**: Consider if SLO is too conservative, invest savings in innovation

Error budgets align incentives between developers (velocity) and SREs (reliability) by making reliability a shared, measurable responsibility.

<br>

<b>SLO Best Practices</b> <br>

- **Document clearly**: Make SLOs visible and understood by all stakeholders
- **Review regularly**: Adjust SLOs quarterly based on user needs and system evolution
- **Monitor actively**: Set up alerts when burning through error budget too quickly
- **Multi-window approach**: Track SLOs over different time windows (hourly, daily, monthly)
- **Don't aim for 100%**: Perfect reliability is impossible and economically wasteful

<br> <br> 

> ### Service Level Agreements (SLAs)
<br>

Service Level Agreements are explicit or implicit contracts between a service provider and customers that include specific consequences for meeting (or missing) the SLOs they contain. While SLOs are internal reliability targets, SLAs are external business commitments with financial or contractual implications.

<br>

<b>Key Differences: SLO vs SLA</b> <br>

**SLO (Service Level Objective)**:
- Internal reliability target
- No direct consequences for missing
- Used to guide engineering priorities
- Can be more aggressive than SLA

**SLA (Service Level Agreement)**:
- External business contract
- Financial penalties or service credits for violations
- Legally binding commitment to customers
- Typically more conservative than SLO to provide buffer

**Golden rule**: Set your **SLO stricter than your SLA** to have a safety margin. For example:
- Internal SLO: 99.95% uptime
- External SLA: 99.9% uptime
- Buffer: 0.05% protects against financial penalties

<br>

<b>Common SLA Components</b> <br>

**Reliability commitments**: Specific uptime or performance guarantees
- Example: "99.9% monthly uptime for the API service"

**Measurement methodology**: How reliability is calculated
- Example: "Measured as successful responses / total requests, excluding scheduled maintenance"

**Exclusions**: What doesn't count against the SLA
- Example: "Downtime due to customer configuration errors, scheduled maintenance windows, or force majeure events"

**Consequences**: Penalties for SLA violations
- Example: "10% service credit for 99-99.9% uptime, 25% credit for <99% uptime"

**Credits/remedies**: How customers are compensated
- Example: "Service credits applied to next month's invoice, maximum 100% of monthly fee"

<br>

<b>Real-World SLA Examples</b> <br>

**Cloud Infrastructure Provider (AWS, GCP, Azure)**:
- **SLA**: 99.99% monthly uptime for compute instances
- **Credit**: 10% credit for 99-99.99%, 30% for 95-99%, 100% for <95%
- **Exclusions**: Scheduled maintenance, customer misconfigurations, external network issues

**SaaS Application (Salesforce, Slack, Zoom)**:
- **SLA**: 99.9% monthly availability
- **Credit**: 10% for 99.0-99.9%, 25% for 95-99%, 100% for <95%
- **Measurement**: Minutes of downtime per month, excluding maintenance windows

**API Service Provider (Stripe, Twilio)**:
- **SLA**: 99.99% API uptime
- **Credit**: Tiered credits based on actual uptime percentage
- **Measurement**: Successful vs failed API requests, measured via monitoring systems

**Managed Database (MongoDB Atlas, Amazon RDS)**:
- **SLA**: 99.995% monthly uptime (Multi-AZ deployments)
- **Credit**: 10% for 99.99-99.995%, 25% for 99.0-99.99%, 100% for <99.0%
- **Measurement**: Database availability across redundant zones

<br>

<b>SLA Best Practices</b> <br>

**For Service Providers**:
- Keep SLA slightly looser than internal SLO (buffer for safety)
- Be specific about measurement methods and exclusions
- Make credit claim process simple and automatic when possible
- Review SLAs annually based on system reliability improvements
- Don't over-promise – violating SLAs damages trust and finances

**For Service Consumers**:
- Understand the fine print (measurement windows, exclusions, claim process)
- Calculate the maximum potential credit (often capped at monthly fee)
- Recognize that credits don't compensate for business impact of outages
- Don't rely solely on SLAs – design for failure with redundancy and fallbacks
- Monitor vendor SLA compliance independently when critical

<br>

<b>The SLI → SLO → SLA Hierarchy</b> <br>

1. **SLI (Indicator)**: Availability is measured as successful requests / total requests
2. **SLO (Objective)**: Internal target of 99.95% availability (21 min downtime/month allowed)
3. **SLA (Agreement)**: External commitment of 99.9% availability (43 min downtime/month) with 10% credit if violated

This hierarchy ensures you have internal warning signals (SLO violations) before hitting contractual penalties (SLA violations).

<br> <br> 

> ## Four Golden Signals
<br>

The Four Golden Signals are a monitoring framework from Google's SRE book that focuses on the most critical metrics for understanding system health. If you can only measure four metrics of your user-facing system, these are the ones to choose. Together, they provide a comprehensive view of system performance and user experience.

<br>
<br>

<b>1. Latency</b> <br>

The time it takes to service a request, measured from the user's perspective. It's critical to distinguish between the latency of successful requests and failed requests.

**Why it matters**: Even if your system is available, high latency creates a poor user experience. Users abandon slow applications.

**What to measure**:
- Successful request latency (HTTP 200): The time users actually experience
- Failed request latency (HTTP 4xx, 5xx): May be very fast (immediate failures) which can skew averages
- Use percentiles (p50, p95, p99) not just averages – p99 shows worst-case user experience

**Example thresholds**:
- Web API: p95 < 200ms, p99 < 500ms
- Database query: p95 < 50ms, p99 < 100ms
- Batch processing: p95 < 5 seconds per job

**Key insight**: Track successful vs failed request latency separately. A spike in 500 errors that fail instantly will make your average latency look good while users are suffering.

<br>

<b>2. Traffic</b> <br>

A measurement of how much demand is being placed on your system, measured in system-specific high-level metrics that matter to your business.

**Why it matters**: Traffic patterns help you understand capacity needs, detect anomalies, and predict when to scale.

**What to measure** (varies by system type):
- **Web service**: HTTP requests per second (RPS)
- **Database**: Queries per second, transactions per second
- **Streaming service**: Concurrent connections, video streams per second
- **IoT platform**: Messages per second, devices connected
- **Storage system**: I/O operations per second (IOPS), throughput in MB/s

**Example thresholds**:
- API service: 10,000 RPS normal, 50,000 RPS peak capacity
- Video platform: 100,000 concurrent streams
- Database: 5,000 queries/second

**Key insight**: Traffic patterns reveal both problems (sudden drop may indicate outage) and opportunities (growth trends inform capacity planning).

<br>

<b>3. Errors</b> <br>

The rate at which requests fail, either explicitly (protocol-level failures) or implicitly (semantically incorrect responses).

**Why it matters**: Errors directly impact user experience and availability. Error rate is inversely related to availability.

**What to measure**:
- **Explicit errors**: HTTP 500s, connection failures, timeouts
- **Implicit errors**: HTTP 200 with wrong content, silent data corruption
- **Client-side errors**: HTTP 4xx (may indicate client problems or API design issues)

**Formula**: Availability ≈ (Total Requests - Errors) / Total Requests

**Example thresholds**:
- Error rate < 0.1% for critical services (99.9% success rate)
- 5xx errors < 0.01% (internal failures)
- 4xx errors tracked separately (often client issues)

**Key insight**: Track errors by type and cause. A spike in 503 (service unavailable) means different action than 500 (internal error). Monitor both the rate and absolute count – 1% error rate at low traffic is different than 1% at high traffic.

<br>

<b>4. Saturation</b> <br>

How "full" your service is – a measure of system resource utilization, emphasizing the most constrained resource.

**Why it matters**: Saturation predicts future problems. Systems degrade non-linearly as they approach capacity (e.g., high CPU increases latency, full disk causes failures).

**What to measure** (focus on bottleneck resources):
- **CPU utilization**: % of CPU capacity used
- **Memory usage**: % of memory consumed, swap activity
- **Disk I/O**: % of I/O capacity, queue depth
- **Network bandwidth**: % of network capacity used
- **Connection pools**: % of database connections in use
- **Queue depth**: Requests waiting to be processed

**Example thresholds**:
- CPU: Alert at 70%, critical at 85%
- Memory: Alert at 80%, critical at 90%
- Disk: Alert at 75%, critical at 85%
- Connection pool: Alert at 80% utilization

**Key insight**: Saturation is a leading indicator – it predicts future latency and errors before users experience them. Different systems have different bottlenecks; measure what constrains YOUR system.

<br>

<b>Why These Four Signals?</b> <br>

**Comprehensive coverage**: Together they answer critical questions:
- **Latency**: Is the system fast enough?
- **Traffic**: How much demand is there?
- **Errors**: Is the system working correctly?
- **Saturation**: Is the system about to break?

**Actionable**: Each signal maps to specific remediation:
- High latency → Optimize code, add caching, scale up
- Traffic spike → Scale horizontally, enable rate limiting
- Error spike → Fix bugs, improve error handling, rollback
- High saturation → Add capacity, optimize resource usage

**User-focused**: These metrics correlate directly with user experience, not just internal system health.

<br>

<b>Practical Implementation</b> <br>

1. **Dashboard**: Create a single dashboard with all four signals for quick health assessment
2. **Alerting**: Set alerts on all four – each predicts different failure modes
3. **Correlation**: Look at all four together during incidents (e.g., saturation spike → latency increase → errors)
4. **Evolution**: Start with these four, then add domain-specific metrics as needed

The Four Golden Signals provide 80% of monitoring value with 20% of the effort – they're the essential foundation for any monitoring strategy.

<br>

> ### Objectives and Key Results (OKRs)
<br>

OKRs (Objectives and Key Results) are a goal-setting framework used to define and track measurable objectives and their outcomes. Originally developed at Intel and popularized by Google, OKRs help align teams around ambitious goals while providing clear metrics for success.

<br>
<br>

<b>Objectives</b> <br>

The qualitative, inspirational goal – what you want to accomplish. Objectives should be:

**Significant**: Meaningful impact on the business or product
**Concrete**: Clear and well-defined, not vague
**Action-oriented**: Inspirational and motivating to the team
**Aspirational**: Stretch goals that push boundaries (typically 60-70% achievable)

**Good objective examples**:
- "Deliver a blazing fast user experience"
- "Become the most reliable payment platform in the industry"
- "Build a world-class incident response capability"

**Poor objective examples**:
- "Improve things" (too vague)
- "Reduce latency by 20ms" (too specific – this is a key result, not an objective)
- "Maintain current systems" (not aspirational)

<br>

<b>Key Results</b> <br>

The quantitative measurements that track progress toward the objective – how you measure success. Key results should be:

**Specific**: Clearly defined and unambiguous
**Measurable**: Quantified with numbers (%, count, time, etc.)
**Time-bound**: Defined timeframe (usually quarterly)
**Aggressive yet realistic**: Challenging but achievable
**Limited**: Typically 3-5 key results per objective (not more)

**Formula**: Each objective should have 2-5 key results that answer "How will we know we've achieved this?"

<br>

<b>Real-World SRE OKR Examples</b> <br>

**Objective 1: Deliver a blazing fast user experience**
- KR1: Reduce p99 API latency from 500ms to 200ms
- KR2: Achieve 95% of page loads under 2 seconds (up from 85%)
- KR3: Decrease time-to-first-byte (TTFB) from 300ms to 150ms

**Objective 2: Become the most reliable payment platform**
- KR1: Increase payment API availability from 99.9% to 99.95% (SLO improvement)
- KR2: Reduce mean time to recovery (MTTR) from 45 minutes to 15 minutes
- KR3: Achieve zero payment data loss incidents (currently averaging 2/quarter)

**Objective 3: Build world-class incident response capability**
- KR1: Reduce time to detect incidents from 10 minutes to 3 minutes (via improved monitoring)
- KR2: Train 100% of on-call engineers on incident command system
- KR3: Conduct 12 disaster recovery drills (1 per month) with 90%+ success rate

**Objective 4: Eliminate toil and scale engineering productivity**
- KR1: Reduce manual toil from 45% to 30% of team time through automation
- KR2: Automate 20 repetitive operational tasks (currently manual)
- KR3: Achieve 90% of deployments fully automated (up from 60%)

**Objective 5: Improve observability and system understanding**
- KR1: Instrument 100% of critical services with distributed tracing
- KR2: Create SLIs and SLOs for all 15 production services
- KR3: Reduce time to root cause from 60 minutes to 20 minutes (via better tooling)

<br>

<b>OKR Best Practices</b> <br>

**Setting OKRs**:
- Align with company/organizational goals
- Collaborate with team – don't dictate from top-down
- Make them public and transparent across the organization
- Set both bottom-up (team-driven) and top-down (leadership-driven) OKRs
- Aim for 60-70% achievement – if you hit 100%, you're not being ambitious enough

**Tracking OKRs**:
- Review progress weekly or bi-weekly
- Grade OKRs at the end of the quarter (0.0 to 1.0 scale)
- Typical scoring: 0.6-0.7 is good (stretch achieved), 1.0 means not ambitious enough, <0.4 needs analysis
- Update key results based on actual data, not estimates

**Common pitfalls**:
- Too many OKRs (focus on 3-5 max per team per quarter)
- Key results that are activities, not outcomes (e.g., "Implement monitoring" vs "Reduce MTTR by 50%")
- Setting OKRs and forgetting them until quarter-end
- Treating OKRs as commitments rather than aspirational goals
- Making them too easy to achieve (should feel uncomfortable)

<br>

<b>OKRs vs Other Goal Frameworks</b> <br>

**OKRs vs KPIs (Key Performance Indicators)**:
- KPIs: Ongoing metrics that measure business health (monitored continuously)
- OKRs: Time-bound goals to achieve specific improvements (quarterly cycles)
- Example KPI: "Maintain 99.9% uptime" (steady-state)
- Example OKR: "Improve uptime from 99.9% to 99.95%" (improvement goal)

**OKRs vs SLOs**:
- SLOs: Service reliability targets (technical commitments)
- OKRs: Business and team objectives (includes but broader than SLOs)
- SLOs can be key results in OKRs (e.g., "Achieve 99.95% SLO for checkout API")

The power of OKRs lies in creating alignment, focus, and measurable accountability while encouraging teams to set ambitious goals that drive meaningful progress.

<br>

