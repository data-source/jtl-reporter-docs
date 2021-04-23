---
title: Virtual Users vs RPS
author: Luděk Nový
author_url: https://ludeknovy.tech
hide_table_of_contents: false
---

Recently I spotted that some people are confused about the requests per second metric and its correlation to the virtual users. Namely, I was asked how is it possible that 10 VU are generating 15.4 RPS, where is the decimal coming from? I did not realize it at first, but when I started playing with performance testing tools a few years ago I was puzzling it over as well. So I would like to use this blog post as an opportunity to shed a bit of light on it and help newcomers in the performance testing domain to properly analyze test performance report. Let's dive into it.
<!--truncate-->


### What is RPS (requests per second)?
RPS is one of the essential performance metrics. It is quite often referred to as throughput, and I like this term better, as it is self-explanatory. It simply tells us how many requests per second the system under test can handle. The throughput is calculated as the total of requests within an interval. Commonly, performance testing tools give us some kind of live preview of currently running tests, the output metrics are calculated within a certain interval. For example, imagine we use a load testing tool and it reports the metrics every 15 seconds. Let’s say we have a running test and for the last 15 seconds, it sent and received 231 requests during this interval. The reported RPS for the last interval would be 15.4 RPS because 231 (total number of the requests) / 15 (interval) equals 15.4 RPS.
This does not mean the tested system maintained the 15.4 RPS consistently within the whole 15 seconds interval. During that period, the RPS could have been changing and reach a higher RPS than the stated 15.4 RPS. Of course, the same logic applies to the ended test as well. But I admit it might look odd to see decimals in RPS for the first time. Just keep in mind, although these metrics might look like actual numbers, they are usually calculated for an interval longer than one second. 

As stated before, the RPS is a crucial metric that reveals fundamental information about the tested system. This metric should only be used in correlation to any response time metrics (e.g.: 90 percentile, 95 percentile, and so on). Every system has a critical point in terms of throughput. Once the system reaches the critical point, it might keep the RPS around the same level, but the response times go up. For this reason, the 15RPS while the 90 percentile equals 5ms means something very different when compared to 15RPS while the 90 percentile response time equals 20ms. For this reason, it is important to conduct multiple test runs. And here the virtual users come into play.


### What are virtual users ?
A virtual user mimics the expected client behavior of the tested system - this could be any client, e.g.: end-user of a web application or a backend application talking to a database, and so on. The typical virtual user life cycle looks like this - it picks up the scripted tasks and performs them, once it finishes all of them it will start the loop again. And it keeps it going until the test is terminated. It is important to realize that the tasks are not performed at the same time by all virtual users. This is determined by several things such as scripted wait times between tasks, actual response times of the application, ramp-up time settings (virtual user spawn rate), hardware utilization of the host, to name a few. And as a result, the load generated by the virtual users gets spread across. To illustrate this let's use an example - imagine you and your friend will pretend a load test using your mobile phones. The scenario would be the following: 1) opening the homepage, 2) opening the first item, 3) adding it into a cart. If you try to do it you realize you are not performing the actions at the same time. The application response time will be different, the browser rendering time will vary as well. Thus, one of you might finish the above-mentioned scenario in 5 seconds, but to the other one, it might take 7 seconds. Imagine you are doing another iteration. This time your phone freezes for a while for some reason, and it takes you 10 seconds to finish the scenario. While your friend finishes it only in 6 seconds. The virtual users behave in the same way. For this reason, we should incorporate wait/think time in our performance scenario to mimic this kind of arbitrariness. As a result, the load generated by virtual users gets spread across in time.


### How does the number of  virtual users impact the RPS?
The number of virtual users affects how the system under test will be loaded, but the number of set virtual users does not mean that they will generate the same amount of RPS because the user load gets spread across as we know from the previous paragraph. The number of virtual users impacts the throughput of a system under test. Let's imagine we do load testing of an e-shop application now. Generating only 50 virtual users would result in 100 RPS and the 90 percentile response time is 8ms. In the second run, we decide to increase the number of virtual users to 500. Now the RPS is 500 and the 90 percentile kept the same response time. This means we did not reach the critical point yet, as the application scales its performance based on the demands so far. Let's do another test with 750VU. This time the measured RPS is 530 and the 90 percentile increased to 30ms. We reached a critical point for the throughput of the tested system. From now on if we keep the number of virtual users increasing, the response times will increase accordingly. If we would run another test with 1500VU, we could probably see very slow response times and some errors due to them as the system would be overwhelmed by the number of open connections and the number of requests it needs to process.

I hope it is a bit more clear now how the virtual users and some other factors impact the throughput and how to interpret the outcomes of the performance testing measurements.