# BOSS: Automatically Identifying Performance Bottlenecks through Big Data

Ruixuan Hou January 11, 2017

[link](https://engineering.linkedin.com/blog/2017/01/boss--automatically-identifying-performance-bottlenecks-through-)

## Introduction
As the centralized performance team of LinkedIn, our mission is to make LinkedIn pages load faster. We help each engineering team try to hit their page load time goals through various optimization efforts. One common question we need to answer when trying to decrease page load time is: where is the performance bottleneck? In other words, where should the engineers focus their efforts? Usually, to answer this question, a performance engineer will look into performance metrics and check some samples captured by Resource Timing API and Call Graph and locate the hotspots. This approach can be very useful, but had the drawback of “trial and error.” Also, many sample waterfalls have to be clicked and analyzed manually to find bottlenecks. We wanted a systematic way that a tool could automatically provide bottleneck details quickly based on existing, large amounts of data.

In this blog, we’ll discuss BOSS (BOttlenecks for Site Speed), a system we built at LinkedIn that analyzes millions of waterfall samples and automatically identifies bottlenecks for improving performance.

## Bottleneck analysis is hard
There are a couple of problems with “manual” bottleneck analysis.

Dealing with multiple performance data sources: multiple systems serve the user’s request, and performance data is tracked separately. We have browser-side measurements using Navigation Timing, Resource Timing API, measurements from native applications (iOS, Android), and server-side tracking data like Call Graph. Each data source has its own unique schema, which makes it difficult to process all of them in one place.

Handling a large volume of performance data: it is important to analyze 100% of our traffic to find the most important bottlenecks. Usually, to find the bottleneck of a page, a performance engineer can look into some samples, and identify some hotspots, but not all. This means that it is easy to miss real bottlenecks and to possibly focus on wrong projects. We wanted to analyze all the data to make sure every LinkedIn member is happy with our site speed. Therefore, we needed to make sure the system could process 100M+ records per day.

Quantifying paralleled calls: finding a bottleneck is not as easy as simply finding the longest request in a waterfall since if there are other calls in parallel, just fixing the longest request can’t reduce the page load time. We needed a model to take both call duration and parallelization into consideration.

Interpreting the performance metrics: there is a lot of domain-specific terminology in performance data, e.g., DNS connection time, redirect time, client render time, etc. It is not easy for developers to understand at first glance why is a page slow. Instead of showing the raw metrics, we wanted to provide actionable items in the result. Examples could include fixing the high response time of your frontend server, removing HTTP redirects in your pages, parallelizing third party request with other calls, etc.

In the following sections, we will explain how we addressed these challenges through BOSS.

## Call tree model to unify performance data sources
The toughest part of automating the analysis is putting various data sources together. We have performance tracking data on both the client side and server side. Those datasets are located separately and have different schemas. To resolve this, we built a generic call tree model to glue data together.

One click from the end user will result in multiple requests to multiple systems. As illustrated below, one typical page view contains API requests to our data center, image/JS/CSS requests to CDNs, and some requests to third parties like Ads. Those requests spread out to multiple systems, and we need a way to trace them in one place.

BOSS1
What does this look like? A tree!

At LinkedIn, we’ve already built call trees between different services, which are found inside our data centers. If we apply this concept on other systems like CDN, third party ads, browsers, etc., we get a bigger call tree.

## Simplifying client-side waterfall
We use Resource Timing data to build the client-side call tree model. However, the raw waterfall contains many page-level navigation timing metrics, like redirect duration, first byte time, page download time, etc., as well as more than a hundred resource timing entries for all the downloads associated with the page—HTTP calls with varying URLs and resource types. This makes it hard to determine the cause of slowness. Bottlenecks need to be actionable. For example, if profile pictures are generally slow to download, it means our media CDN needs to be investigated, rather than everyone’s profile pictures.

To solve this problem, we came up with bottleneck types that we assigned to each resource/metric in the waterfall in order to get actionable bottlenecks.

Bottleneck Type	Resource Timing/Navigation Timing data source
Server-side	Time to first byte and content download time of request to linkedin.com domain
CDN	Request to our CDN domain
Long native code execution time (JS/parsing, rendering) on client	Gaps in waterfall and customized markers using user timing data
Third-party content	Request not served by LinkedIn-owned domains
Redirect	Redirect time and count of navigation timing data
Note that we saw a lot of gaps in waterfalls where no network activities happened. After debugging locally, we found that there is a lot of heavy native code execution (js/parsing/rendering) during these gaps. To measure this more precisely, we started using user timing API to measure the key rendering paths so that we get more insight.

## Call trees analysis
Once we have the “combined tree,” the next challenge is how to analyze these trees to find the performance bottlenecks of each page. Basically, there are two kinds of calls that hurt performance:

## Slow calls

## Sequential/blocking calls

Developers are very sensitive to the latency of individual requests, but sometimes neglect the importance of parallelization of calls. Let’s use two hypothetical pageviews as example. The first one parallelizes the CSS and JS calls with the HTML request, but the second one does not. The latency for all calls is the same, but the page load time has 1,100ms difference! The bottleneck here is that the HTML request is blocking the CSS and JS requests.

BOSS penalizes both slow calls and blocking calls. In the example above, our bottleneck analysis will give the HTML 38.9% for parallelled page view and 72.2% for unparallelled page view. Even if the duration of each call is the same, the blocking HTML call gets more penalties in our analysis and is marked as the bottleneck of the page. The algorithm we use is based on existing service contribution tools, and looks into the call tree between services. Given that client-side data also fits into our generic call tree model well, and we now have a “combined call tree,” we can apply heuristics from service contribution to the client side.

To understand the algorithm that calculates the bottleneck contribution, let’s use a simplified version of page view call tree. The timeline is divided into different segments based on the start and end duration of each call. For each time segment, if there are multiple calls happening in parallel, we will contribute this segment evenly to every call. In the example below, the 90ms segment is divided into 3 pieces and each service gets assigned a 30ms contribution. For the 100ms segment of Call A, there are no other calls in parallel and it gets blamed for the whole time segment, which will result in a very high contribution for Call A. Calls with high contributions will be the top bottlenecks. As a result, Call A gets 195ms, which is 60.9% of the total page load time of 320ms, due to the first 100ms blocking segment plus its long duration.  Call B gets 20.3% and Call C gets 18.8%, since they are well-paralleled with each other.

## Analyzing performance bottlenecks at scale
Processing the call tree data is non-trivial. We are getting millions of page view records and each record creates a call tree with hundreds of nodes. Here are the requirements of our data processing system and the solutions we picked:

Scalable to massive amount of data: we picked a Kafka + Hadoop solution to handle the data, which has proven to be very successful at LinkedIn.
Ability to slice and dice data into different dimensions and aggregated metrics: what is the bottleneck for the slowest 10% of members? What is the bottleneck for members using a 4G Network? What is the bottleneck for members who only visit our website once a week? We picked Apache Pig as the language of choice to perform these tasks in Hadoop.
Fast iterations on call tree analysis algorithms: the logic of processing a call tree is complicated and needs to be tuned in multiple iterations. It is impractical to use Pig to handle the logic and test it in Hadoop every time. To solve this, we use UDFs (user defined functions) written in Java/Python to handle the call tree analysis logic and write unit tests for fast iterations.

With the scalable data analysis system available, the next challenge is building an algorithm to analyze the call tree.

UI for bottleneck analysis: putting it all together
On the UI side, we built the following components to assist the analysis:

Bar chart and pie charts to highlight top bottlenecks;
Stacked trending chart to show how the bottleneck changes over time;
A table for easy sorting and look up.

On the same UI, users can also view the latency distribution of page views in a scatter plot and click each point to get the full waterfall of the page.

With this powerful tool, users can easily find the bottlenecks with a few clicks.

## Real-world example
Here is a bottleneck analysis we ran for our LinkedIn desktop home page in March 2016. Each type of bottleneck has a contribution ratio, which means “by removing this bottleneck, how much site speed improvement we can have.” Here is the table that lists top bottlenecks and the solutions to fix each.

Bottleneck Type	Contribution Ratio (%)	How to Fix
Server response	27.32 %	Server-side bottleneck mainly comes from slower services
Gaps without network activities	22.16 %	This bottleneck indicates some JS execution or browser parsing and rendering are blocking other network requests
CDN objects	20.16 %	This bottleneck indicates slower CDN vendor or big image/JS/CSS size
AJAX calls	9.21 %	This can be fixed by either consolidating the AJAX call with the HTML response or deferring it until after page load
Ads calls	7.57 %	Ads calls should not be blocking the following calls
Redirects	4.50 %	Unnecessary redirects should be avoided; they are always blocking the following request
Network connection	3.65 %	This indicates slow TCP handshake connections, a problem that can come from the local ISP provider or our proxy server
After finding the bottlenecks, the next step was locating the particular requests/code causing the bottlenecks. We examined a couple of waterfalls and found that a long gap happens frequently after downloading the ads. That indicates that there are heavy JavaScript executions after downloading the ads and the next requests are waiting for the JavaScript to finish execution.

The fix was easy: we just made the ads load in an unblocking way so that images and other things can be loaded at the same. After running A/B testing, it turned out that our home page became 21% faster in page load time. At the same time, we’ve seen a boost in engagement metrics; after deferring ads, users are more engaged with the site.

And in our bottleneck analysis data, the contribution of gaps and ads calls dropped significantly.

## Conclusion
We have built a bottleneck analysis tool, BOSS, that can process data at scale and produce actionable optimization items. We already have several success stories so far but plan to make additional improvements, including:

More focus on server-side analysis. So far, BOSS works well for client-side analysis. We want to do the same on the server side. For example, bottleneck analysis for different API endpoints, service calls across different data centers, etc.

Auto suggestion for performance optimization. Currently, the tool is used in a passive mode: analysis is only done when a user visits our tool and wants to do some optimization. Instead, we want to automatically run analysis for every page and send optimization suggestions directly to page owners. In this way, we can bubble up performance issues easily and make improvements early.

## Acknowledgements
Thanks to David He and Ritesh Maheshwari for the invaluable input and feedback. Thanks to Toon Sripatanaskul for his awesome service contribution algorithm. Thanks to Swapnil Ghike and Joseph Zemek for their pioneering efforts for this project. And thanks to Oliver Tse for being our avid user and for his valuable feedback. And finally, thanks to Steven Pham and Dylan Harris for their help in developing the BOSS UI.
