In my job role, I get an opportunity to work with various shiny storages available in the market. Sharing some tips in this document which I use to evaluate any storage from performance angle. Disclaimer: I am not a performance engineer. Like all other infra engineers I also wear multiple hats in my job role. Typically in a month for a day or two that hat is performance engineering. 

On a broader note: This is a general sequence

1) Understand the architecture of storage
2) Calculate the overall network throughput. Found the chokepoint, this chokepoint is a benchmark figure for us. 
3) Barebone testing with standard tuning options from vendor white papers or the vendor recommendations to get the max throughput. (Record the results)
4) Actual workload testing. (Record the results)

Usually people gets excited about the various tuning options. I rarely see a hidden knob (not available in any official vendor doc) which can significantly improve the performance. May be sometime something at the cost of reliability - big no for prod workloads. 


## Understand architecture

### Commodity vs properitry hardware 

#### Commodity

Storage vendors only sell the software layer and have list of certified vendors with HW models which companies can purchase to deploy the storage vendor software layer. 

#### Properitary

Storage vendors sell both hardware and software layer. 

### Shared nothing vs Shared everything

Very common terms used in storage industry, some vendors come up with new terms to grab customer attention. NFS is divine, may be some people don't like this statement but NFS still rule the industry. 

#### Shared Everything

Usually in this architecture storage and compute are decoupled. In theory storage and compute can be scaled independently but at scale vendors may start coming up with recommendation of keeping some ratio between compute and storage boxes otherwise one of them will become bottleneck. 

#### Shared nothing

storage and compute is tightly coupled. Adding new boxes in existing storage system provides both. Need to worry less about maintaining ratio. Not a perfect fit for every use case, comes with own set of challenges in mgmt. 

### Storage access protocol

#### Parallel filesystem(storage)

With client dependency it's painful to use but provides better performance in-comparison to NFS based storages. 

#### NFS (or other protocol)

Clients mounting the storage using NFS based protocol. No dependency on client comes at the cost of performance. 

### Storage traffic 

How frontend (N/S) and backend traffic (E/W) is segregaeted 

frontend traffic is the traffic coming from client nodes. 
backend traffic is the traffic between storage nodes.

## Network throughput calculation

In era of NVMe disk and SCM (storage class memory) disks rarely become bottleneck unless you have very small I/O workload. In our environment usually network is a bottleneck. 

Aside note: Intel introduced optane (SCM) in the market, they have discontinued this product. Other vendors like Samsung and Kioxia still actively work in SCM space. 

Sit along with your network team to calculate the max network throughput expected from the architecture. Found the chokepoint - which is least network throughput from any network layer between your clients and storage stack this is your benchmark value. 

Ex: If storage nodes are connected with ToR (Top of Rack) switches with total 1.6Tb/s but ToR switches are connected with core only 800Gb/s then max throughput you may achieve is 800Gb/s. 

This factor helps to decide how much traffic we need to generate from the client nodes to get this max throughput. If you have 40Gb/s client nodes 20-25 such nodes should be sufficient to hit the max throughput of storage assuming clients are not generating any other traffic. 

## Barebone testing

This is the first stage of testing of us where we follow standard tuning recommendation of vendor on storage and client system. If any recommendation conflicts with our client settings we avoid that parameter unless it's must from vendor. 

Before proceeding further make sure that dashboards are prepared. 

Storage dashboards: Some vendors provide good monitoring in storage UI, some provide prometheus compatible exporters (with grafana dashboards) to scrape these metrics.
Client dashboards: Many utiilites are available like sar or prometheus node exporter or custom scripts. 

Since purpose of this testing to get max throughput (touching chokepoint) hence these tests use bigger block sizes like 1M or 4M. Prepare your tests, I personally prefer fio instead of vendor provided performance tools. 

Start running test from a single client and record how much traffic generated from single client. This helps to verify how much throughput fio (or any other tool) able to generate from single client. Increase the number of threads or blocksize to reach the max limit of client NIC. 32 Gb/s (~80%) from 40 Gb/s (NIC card client) is good indicator that we are reaching max limit of client. Also ensure that test run for sufficient duration. 

Once max throughput of client is reached, run this test from multiple clients. Utilities like pdsh can be used to run test from multiple clients, ensure that test run from all clients parallely and it keeps on running, sometime test fails from few clients because of missing package or any othe reason. 

Important: store the performance tool output in form which is easy to parse so that numbers can be plotted. 

Purpose of this test is to give us confidence that we are able to get expected numbers from the storage system. 

## Workload testing

Usually storage systems are not purchased for single application hence after the doing the barebone testing and before doing any particular application workload testing, run test with different blocksizes and increase the number of clients. After all true litmus test for storage system is with small block I/O operation, most storages shine with bigger block sizes. 

Depending upon the complexicity and environment may be for this task application team involvement is needed. We have test suite to reproduce the application workload. We try to simulate the workload 







