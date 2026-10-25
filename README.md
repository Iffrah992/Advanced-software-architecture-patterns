# Advanced-software-architecture-patterns

**ADR(Architecture Decision Record) for the flash sale system**
**1. ADR header**
Field	Value
Project: FLASH SALE SYSTEM
Status: In progress
Date: 25-10-2026
Author(s): Iffrah, Aema, Sumaya
Reviewer: Dr. Adamkó Attila Tamás 
Architecture style	[Layered / Event-driven / Microkernel / Microservices / Space-based]
Repository / branch:


**2. Context and Problem Statement**
**Background:**
This project studies flash-sale systems, where a very large number of users try to buy a very limited number of items at the same moment. The goal is to find where the system starts to fail, identify the real bottlenecks, and improve the architecture based on evidence from testing.

**Scenario:**
Parameter	                    Value
Product	Limited-edition       shoes
Stock available	              100 pairs
Expected buyers              	~10,000
Sale start	                  12:00, all buyers click "Buy" at the same time
Traffic pattern	              Normal on regular days; extreme spike at sale start

The main issue is not only high traffic. All users are competing for the same small stock at the same time, which puts pressure on a few shared resources.

**What happens behind one "Buy" click**
Check stock
Reserve the item
Create the order
Process the payment
Confirm the order

For one user this is easy. When thousands of users run these steps together, parts of the system become overloaded.

**Identified bottlenecks**

**P1**– Database contention (database jam). Every application server must access the same stock record in the inventory database. Adding more application servers does not help; it sends even more requests to the same record, causing waiting and locking. This is like having many entrances but only one checkout counter.

**P2** – Overselling. When only one item is left, two users may both check the stock before either one updates it. Both believe they got the item, and the system sells more than it has.

**P3** – Slow payment dependency. Payment depends on an external service. If the payment service is slow and the system keeps database connections busy while waiting, all connections can eventually be occupied, and new buyers cannot even start a purchase.

**P4**– Retries and duplicate requests. When users wait longer, some click "Buy" again, or the system retries automatically. This adds more traffic and makes the overload worse. Without detection, the same user could be charged twice.

**P5**– Fairness. With 10,000 people competing for 100 items, there must be a clear rule for who gets them. Without one, allocation depends on network speed, retries, or whichever server processes a request first.

**Problem statement**

How should the flash-sale system be structured so that it can handle ~10,000 simultaneous buyers competing for 100 items without database contention, overselling, payment-related resource exhaustion, duplicate charges, or unfair allocation?

**Assumptions and constraints**
The final solution is not fixed yet; architecture decisions will be driven by load-test results.
A basic version of the system will be built first, then tested under heavy load.
Payment is handled by an external service whose response time is outside our control.
Measurements will focus on database locks, connection usage, and payment delays.
Only one part of the architecture will be changed at a time so results can be compared.

Tech stack: [language, framework, database, message broker]

**3. Requirements and Decision Drivers**
Functional requirements
FR1: A buyer can see the sale item and whether it is still available.
FR2: A buyer can reserve one item; the stock check and update happen as one safe operation.
FR3: An order is created for each successful reservation.
FR4: Payment is processed; if it fails, the item is released back on sale.
FR5: The buyer gets a clear result: confirmed, sold out, or payment failed.

**Quality attributes (decision drivers)**
Priority	Quality attribute	Linked problem	Target / measure
1	Correctness (no overselling)	P2	Confirmed orders never exceed 100
2	Scalability under contention	P1	System keeps serving requests during the 12:00 spike: [X requests/sec]
3	Resource efficiency	P3	DB connections are not held while waiting for payment; pool never exhausted
4	Idempotency	P4	A repeated click or retry produces one order and one charge
5	Fairness	P5	Items allocated in the order requests are accepted by the server
6	Responsiveness	P3, P4	Buyer receives a result quickly: [p95 under X ms]
7	Availability	All	System stays up for the whole sale


**4. Considered Options**
Option	Description	Main strength	Main weakness
A. Baseline synchronous design	One request goes through all five steps in sequence and keeps resources busy until payment finishes	Simple to build and understand	DB is busy the whole time, including while waiting for the bank; vulnerable to P1–P5
B. Improved controlled design	Entry is controlled, the item is reserved quickly and safely, DB resources are released, and slow work such as payment is processed separately through a queue	Targets each identified bottleneck directly	More components to build and coordinate


**5. Decision Outcome**
Decision (proposed): We will use Option B, the improved controlled design, as the target design. Option A will be built first as the baseline for load testing. The architecture style used to implement Option B will be evaluated separately in ADR-0002 to ADR-0006.

**Rationale:**
Controlling how many requests enter at once reduces contention on the shared stock record (P1).
Making the stock check and update one safe operation guarantees only one request can take the last item (P2).
Releasing database resources before payment stops slow external payments from exhausting connections (P3).
Giving every request a unique ID lets the system recognise repeated requests, so no one is charged twice (P4).
A server-side queue gives a clear, predictable allocation rule (P5) and lets heavy work be processed gradually.

**Why Option A is not the target design:**
It keeps the database busy during the whole purchase, including the external payment wait.
It has no protection against overselling, retries, duplicate charges, or unfair allocation.
It is kept only as the baseline to measure which bottleneck appears first.
6. Architecture Description

**Diagram:**

**Components**
Component	Responsibility	Technology
Entry gate (queue / rate limiter)	Controls how many requests enter the system at once; orders accepted requests	[ ]
Reservation / inventory service	Performs the safe check-and-take on stock; releases items when payment fails	[ ]
Inventory database	Stores the stock record; busy only during the short reservation step	[ ]
Order queue	Holds reserved orders so slower work is processed gradually	[ ]
Order / payment workers	Create the order and call the external payment service without holding DB connections	[ ]
External payment service	Processes the payment (outside our control)	[Payment provider / mock]
Notification / confirmation	Tells the buyer the result: confirmed, sold out, or payment failed	[ ]
Request flow for one purchase

**Before (baseline):** Click → Check + reserve → Wait for bank → Confirm. The database is busy for the whole flow.

**After (target):**
Click: buyer presses Buy; the request gets a unique request ID.
Gate: the queue / rate limiter lets in a manageable number of requests, in the order the server accepts them.
Reserve: the stock check and update happen as one safe operation. This is the only step where the DB is busy.
Queue: the reserved order is placed on a queue and DB resources are released.
Pay: a worker processes the payment through the external service.
Outcome:
Payment succeeds → order confirmed.
Payment fails → item released back on sale.


**7. How the Design Handles Each Bottleneck**
Problem	 Proposed fix	How it works	Component responsible	Handled well?
P1 Database jam	Controlled entry	Only a manageable number of requests reach the stock record at once	Entry gate	To be confirmed by load test
P2 Overselling	Safe stock claim	Check and take happen as one unbreakable step; the second buyer sees "Sold out"	Reservation service	To be confirmed by load test
P3 Slow payments	Don't make the DB wait	Reserve → release DB → pay; failed payment returns the item	Order / payment workers	To be confirmed by load test
P4 Retries / duplicates	Unique request ID	A repeated click is recognised, so there is no double charge	Entry gate / order service	To be confirmed by load test
Overload	Process gradually	Orders join a queue and workers handle them steadily	Order queue + workers	To be confirmed by load test
P5 Fairness	Server-side queue	First valid request accepted by the server is processed first	Entry gate	To be confirmed by load test

**Fairness policy chosen:** Strict FIFO through a centralised server-side queue. Reason: client click time cannot be trusted, because user devices and network delays differ. Server-side ordering makes allocation more predictable, transparent, and less dependent on random processing order. Further study: compare strict FIFO with randomised queues or lottery-based allocation if fairness requirements change.

**8. Consequences**
**Positive**
No overselling: only one request can take the last item.
The database is busy only for the short reservation step, not during payment.
Repeated clicks and retries no longer create duplicate orders or charges.
Load is smoothed out by the gate and the queue instead of hitting the database all at once.
Allocation follows a clear, explainable rule.
**Negative**
More components (gate, queue, workers) to build, run, and monitor than the baseline.
The buyer may not get the final payment result immediately, because payment is processed separately.
Reserved items must be reliably released when payment fails.


**Risks and mitigations**
Risk	Likelihood	Impact	Mitigation

**Trade-offs accepted**
More system complexity in exchange for correctness (no overselling), protection of database resources, and stable behaviour under extreme load.

**9. Evaluation Matrix**
Filled in ADR-0007 after each architecture style (ADR-0002 to ADR-0006) has been built and tested.

Decision driver	Weight	Layered	Event-driven	Microkernel	Microservices	Space-based
Correctness 
Scalability under contention	
Resource efficiency	
Idempotency	
Fairness
Responsiveness	
Availability	
Weighted total	100%	

**10. Validation and Implementation**
Validation method
The decision is validated with evidence, not guesses, using a repeated loop:

Build simple: build the baseline system (Option A).
Load-test: simulate ~10,000 buyers clicking at 12:00.
Find what breaks: measure which bottleneck appears first.
Change one thing: apply one fix at a time, then test again and compare.

**Suspects to measure:** database locks on the stock record, connection pool usage, payment delays.

**Load test setup:** [Tool, e.g. k6 / JMeter / Locust] · 10,000 simulated buyers · 100 items in stock · [Ramp-up pattern] · [Environment]

**Measurements per test run**
Run	Change made	Main suspect	Throughput (req/s)	p95 latency (ms)	Orders confirmed	Oversold?	Duplicate charges	First thing that broke
1	Baseline (Option A)	DB locks on stock
2	[e.g. add entry gate]	Connection pool	
3	[e.g. separate payment]	Payment delays	
4	


**Success criteria**
Exactly 100 orders confirmed, 0 oversold.
0 duplicate charges.
Database connection pool never fully exhausted.
[p95 response time under X ms]


**Implementation plan**
Build the baseline system and the load-test setup.
Run the baseline test and record which bottleneck appears first.
Apply the fixes one at a time and re-test after each change.
Implement the improved design in each architecture style (ADR-0002 to ADR-0006).
Compare the styles in ADR-0007 and update this ADR's status to Accepted.

**Review date:**

**11. References**
- Richards, M. (2015) *Software Architecture Patterns*. Sebastopol, CA: O'Reilly Media.
- Richards, M. and Ford, N. (2020) *Fundamentals of Software Architecture: An Engineering Approach*. Sebastopol, CA: O'Reilly Media.
