#Real-time distributed tracing for website performance and efficiency optimizations
Cuong Tran February 3, 2015

## Co-authors: Chris Coleman , Toon Sripatanaskul

With LinkedIn’s service-oriented architecture, a single page view request can fan out calls to downstream services spanning multiple backend tiers, many levels deep. Though applications depend on downstream services, developers have no insight on the relationships and performance of these services. This poses a number of major challenges, including performance optimization and root cause analysis.

To have real-time clarity into service performance across tiers, LinkedIn built ubiquitous distributed tracing. For more information see the blog post on Apache Samza . This post describes how inCapacity, an internal LinkedIn tool, consumes results from Samza to build real-time call graphs that profile requests in a distributed architecture; the profile includes the request, associated webservers, and downstream tiers in the data center.

These distributed call graphs have helped unlock a whole new range of very useful tools; we now have tools for performance correlation and root cause analysis of service-oriented applications, efficiency analysis for cost, and resource headroom for capacity planning in the context of the web pages.

## Insights into end-to-end performance and efficiency with inCapacity

Here is an example of a request from a mobile application to a frontend service in the LinkedIn website. Please note that in this and subsequent sections, service names are generic and latency values are dummy values for the discussion purpose only.

This request in turn, fans out service calls to multiple downstream services. As this  blog post on Apache Samza describes, the traces for these services from the same inbound call are grouped and republished as a single event on a message bus for real-time analysis. Requests from the same web page are tagged with the same pagekey, which is analogous to webpage identification, allowing performance and efficiency analysis in the context of the web application rather than the individual cost of a single service; for example, mobile app versus a DB Service.

Using the request ID in the traces, inCapacity gathers traces from the same in-bound call, traverses the call paths, and tracks performance with different dimensions and in the context of the specific application, giving developers insight into:

The services the application depends on and how often they are used for a request from the application
The services that take the majority of time
The APIs used, how often they are called, and ones that stress the service the most
inCapacity reflects the most current view of the infrastructure; because services change regularly, having current information is important to discover changes that help and/or hurt overall performance.

For the purpose of discussion, let’s simplify a call path:

Call path performance: Along each call path of a call graph, inCapacity tracks count and response time of every API call. If an API makes downstream calls, inCapacity factors out wait time for the call; the result is the API self-latency. For example, API_B at Svc_B calls API_C1 and API_C2 in sequence. Self-latency of API_B is the difference between latency of API_B and sum of API_C1 and API_C2 latency from Svc_C. Then, by comparing API self-latency with API total, inCapacity can assess if the total latency of API is dominated by downstreams or by itself. Tracking API calls and performance along the call paths gives web developers insight into their page dependency on service APIs and their performance.
Workload on services. For each service in the call paths, inCapacity aggregates count and latency of inbound calls per page. Because the traces are tagged with pagekeys, inCapacity can group its workload by the originator of requests, that is, the pages or applications, rather than just by the immediate callers. It can then give the service provider insights into the workload of the services in the context of page views.
Note that call paths of a page are aggregated every 15 minutes. The count and latency of an API is thus total and average the latency over 15 minutes.

Besides performance and root cause analysis, call graphs can provide insight into cost efficiency of web pages. For example, we can compute cost of a page as sum of the cost of downstream services evoked by the page. Cost of a service for the page is proportional to the page workload on the service. This is approximated as the total latency from service for the page, that is, the product of call count and call latency.

Other use cases made feasible by call graphs include capacity planning in the context of web pages rather than just an isolated service. For example, if we expect page views for a given page to grow by 20% next quarter, we can determine the hardware provisioning needed for all services that the page depends on. We can also project the performance and cost of a new feature, given the cost and performance details of associated services for the page. This is very useful for cost vs. benefit evaluation before building a feature.

## Performance Analysis with inCapacity
We use a top-down approach for analysis of web page applications, where we drill down from the high-level summary, to call paths, and finally navigate through the randomly-picked call trees.

Summary view shows a birds-eye view of the services that the application depends on and how often they are used. inCapacity shows the most heavily used services first.

Call path view. The entire call path from the initial page view request to each downstream service is displayed. Developers can assess in granular detail, the services and APIs their applications depend on, and how they perform. This comprehensive view can highlight issues downstream that developers are not aware of, such as slow backend storage. inCapacity stack ranks calls at the same call level, showing the slowest call path first. It ranks by API total latency, which is the product of count of calls and average latency.
Here’s a call path for GET /aaa from a browser:


The analysis shows GET /aaa is slow because of the call to Service_C, that is, self-latency of GET /aaa is lower than the wait time. Self-latency is the difference between average latency and wait time for downstream calls. If the self-latency of a service is higher than the wait time for downstreams, this service is the slowest service. Drilling down the call path, we can see the slowest service is Service_F.

When computing self-latency, inCapacity takes parallel downstream calls into consideration. Overlapped wait time of parallel calls is factored out.

Single call graph view. This view displays a randomly sampled single call graph. Developers can navigate and examine the sequence of service calls in the timeline across the data center. With this waterfall view, developers can look for areas of improvements and analyze whether the frequency and count of downstream calls are reasonable, the time lapse between 2 downstream calls is acceptable, and whether parallelizing these calls would be more efficient. inCapacity interfaces with Call Tree viewer, a LinkedIn internal tool for call tree visualization, to provide this view.

inCapacity allows service providers analyze traffic for different use cases:

Performance analysis: Identifies the pages that call a service, APIs used, the frequency of use, and the performance of the service for these pages. It also identifies outbound calls from this service and their performance.
Dependency analysis . Identifies pages that would be affected the most if these APIs were deprecated
Root Cause Analysis with inCapacity
Analyzing root cause of performance issues in a website of LinkedIn’s scale is difficult even in the best of times. inCapacity’s call graph performance comparison feature can take the guesswork out of such analysis.

When comparing the performance of a call graph between two time intervals, inCapacity first stack ranks call paths by their total latency change. It then navigates along the path and at each call level, compares the change in self-latency of the API with the change of wait time for downstream calls by that API. If the change in self-latency is higher, the API is responsible for the call path performance change. Otherwise, the analysis moves to the downstream paths and the logic starts again, top ranking the downstream paths and navigating the top-ranked downstream path.

The figure below shows how we use call graph performance comparison for root cause analysis of a service call to Service_B.

We compare difference (D) of average latency between the baseline and current time.The difference is 24.4 msec and degrades by 73.93%. The difference in self-latency is only 5.6 msec, so we infer that the problem is downstream. Traversing down the call path, the next downstream call is to Service_C and the latency difference at this level is 11.2 msec. Again, self-latency at this level is only 0.3 msec, so we examine the next call, which is Service_D and find the latency difference 11.4 msec is dominated by the self-latency difference. The issue is thus at Service_D. Further performance analysis of Service_D indicates the root cause was maxed out DB sessions.  This use case shows inCapacity can help developers quickly localize the service which is the cause of the performance issue.

## Summary
Real-time distributed tracing and call graphs have proven to be indispensable for insight into the service performance of web applications, and the associated cost across the entire technology stack at LinkedIn. Performance and cost of services are not analyzed in isolation but in the context of their impact on the entire site. This takes the guesswork out of root cause analysis by identifying the long poles in web application performance or applications affected by changes in service such as API deprecation or data center failover. Call graphs also add useful dimensions in cost analysis such as cost per web application. Such insights help us make an educated decision on services to optimize for global performance gain as well as cost effectiveness. Future work will include adding visualization and automation of analysis and server sharing.

## Acknowledgement
We would like to thank the Samza team for the distributed traces and single call graph viewer, Thomas Goetze and Badri Sridharan for their invaluable feedback and support, and Jay Kreps for the conception and building of Kafka and Samza.

Topics

distributed service call graph, root cause analysis, performance analysis
Related story
Who moved my 99th percentile latency?
Related story
Benchmarking Apache Samza: 1.2 million messages per second on a single node
Back to topLinkedIn.com
Blog  Data  Open Source  Jobs  Women in Tech
LinkedIn Corporation © 2017
 About Cookie Policy Privacy Policy User Agreement
