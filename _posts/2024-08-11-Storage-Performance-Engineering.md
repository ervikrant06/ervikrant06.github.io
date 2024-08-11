In my job role, I have the opportunity to work with various storage solutions available in the market. I’m sharing some tips in this document that I use to evaluate any storage from a performance angle. Disclaimer: I am not a performance engineer. Like all other infrastructure engineers, I also wear multiple hats in my job role. Typically, for a day or two each month, that hat is performance engineering.

On a broader note: This is a general sequence.

* Understand the architecture of storage.
* Calculate the overall network throughput. Identify the chokepoint; this chokepoint is a benchmark figure for us.
* Barebone testing with standard tuning options from vendor white papers or recommendations to get the maximum throughput. (Record the results.)
* Actual workload testing. (Record the results.)

Usually, people get excited about the various tuning options. I rarely see a hidden knob (not available in any official vendor documentation) that can significantly improve performance. Maybe sometimes something at the cost of reliability—a big no for production workloads.

## Understand Architecture

#### Commodity vs Proprietary Hardware

Commodity
Storage vendors only sell the software layer and provide a list of certified hardware vendors and models that companies can purchase to deploy the storage vendor's software layer.

Proprietary
Storage vendors sell both hardware and software layers.

#### Shared Nothing vs Shared Everything

These are very common terms used in the storage industry; some vendors come up with new terms to grab customer attention. NFS is significant; some people might not like this statement, but NFS still rules the industry.

Shared Everything
In this architecture, storage and compute are usually decoupled. In theory, storage and compute can be scaled independently, but at scale, vendors may start recommending a specific ratio between compute and storage boxes; otherwise, one of them will become a bottleneck.

Shared Nothing
Storage and compute are tightly coupled. Adding new boxes to an existing storage system provides both. You need to worry less about maintaining a ratio. This is not a perfect fit for every use case and comes with its own set of management challenges.

#### Storage Access Protocol

Parallel Filesystem (Storage)
With client dependency, it's painful to use but provides better performance compared to NFS-based storage.

NFS (or Other Protocol)
Clients mount the storage using an NFS-based protocol. There is no dependency on the client, but this comes at the cost of performance.

#### Storage Traffic

How frontend (N/S) and backend traffic (E/W) is segregated:

Frontend Traffic: Traffic coming from client nodes.
Backend Traffic: Traffic between storage nodes.

#### Network Throughput Calculation

In the era of NVMe disks and SCM (Storage Class Memory), disks rarely become a bottleneck unless you have very small I/O workloads. In our environment, the network is usually a bottleneck.

Aside Note: Intel introduced Optane (SCM) in the market, but they have discontinued this product. Other vendors like Samsung and Kioxia still actively work in the SCM space.

Sit with your network team to calculate the maximum network throughput expected from the architecture. Identify the chokepoint— the least network throughput from any network layer between your clients and the storage stack. This is your benchmark value.

For example: If storage nodes are connected with ToR (Top of Rack) switches with a total of 1.6 Tb/s, but ToR switches are connected to the core with only 800 Gb/s, then the maximum throughput you may achieve is 800 Gb/s.

This factor helps determine how much traffic you need to generate from the client nodes to reach this maximum throughput. If you have 40 Gb/s client nodes, 20-25 such nodes should be sufficient to hit the maximum throughput of storage, assuming clients are not generating any other traffic.

## Barebone Testing

This is the first stage of testing where we follow standard tuning recommendations from the vendor on storage and client systems. If any recommendation conflicts with our client settings, we avoid that parameter unless it’s a must from the vendor.

Before proceeding further, make sure that dashboards are prepared.

Storage Dashboards: Some vendors provide good monitoring in storage UI, while others offer Prometheus-compatible exporters (with Grafana dashboards) to scrape these metrics.
Client Dashboards: Many utilities are available, such as sar, Prometheus node exporter, or custom scripts.
Since the purpose of this testing is to get the maximum throughput (touching the chokepoint), these tests use larger block sizes like 1M or 4M. Prepare your tests; I personally prefer fio over vendor-provided performance tools.

Start running tests from a single client and record how much traffic is generated from that client. This helps verify how much throughput fio (or any other tool) is able to generate from a single client. Increase the number of threads or block size to reach the maximum limit of the client NIC. 32 Gb/s (~80%) from a 40 Gb/s (NIC card client) is a good indicator that we are reaching the client’s maximum limit. Also, ensure that the test runs for a sufficient duration.

Once the maximum throughput of the client is reached, run this test from multiple clients. Utilities like pdsh can be used to run tests from multiple clients. Ensure that tests run in parallel from all clients and continue running; sometimes, tests fail from a few clients due to missing packages or other reasons.

Important: Store the performance tool output in a form that is easy to parse so that numbers can be plotted.

The purpose of this test is to give us confidence that we are able to achieve the expected numbers from the storage system.

## Workload Testing

Usually, storage systems are not purchased for a single application. Hence, after doing the barebone testing and before conducting any specific application workload testing, run tests with different block sizes and increase the number of clients. After all, the true litmus test for a storage system is with small block I/O operations; most storages excel with larger block sizes. There’s no harm in running the tests to find the shortcomings of storage, but that shouldn’t be the objective. It should be considered as points to keep in mind for future workloads. Stick more to your majority of actual workload testing.

Depending on the environment, application team involvement might be needed to run actual application workload testing. We have a test suite to reproduce the application workload. We try to simulate the current workload and future anticipated workload of the application(s) while evaluating storage.

Record and plot the results. 

## Summary

Good luck with your storage purchases :-)

Why "purchases" (not "purchase"): From my personal experience, no silver bullet storage solution is available in the market to suit all workloads. You may have to purchase multiple storage systems (but that doesn’t mean new storage for each app).

Ending this article with a favorite quote about storage:

“You can buy your way out of bandwidth problems. But latency is divine.” - Mark Seager
