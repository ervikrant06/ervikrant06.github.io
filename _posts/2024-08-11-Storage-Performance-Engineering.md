In my job role, I get an opportunity to work with various shiny storages available in the market. Sharing some tips in this document which I use to evaluate any storage from performance angle. Disclaimer: I am not a performance engineer. Like all other infra engineers I also wear multiple hats in my job role. Typically in a month for a day or two that hat is performance engineering. 

On a broader note: This is a general sequence

1) Understand the architecture of storage
2) Calculate the overall network throughput. Found the chokepoint, this chokepoint is a benchmark figure for us. 
3) Barebone testing with standard tuning options from vendor white papers or the vendor recommendations to get the max throughput. (Record the results)
4) Actual workload testing. (Record the results)

Usually people gets excited about the various tuning options. I rarely see a hidden knob (not available in any official vendor doc) which can significantly improve the performance. May be sometime something at the cost of reliability - big no for prod workloads. 


## Understand architecture

Shared nothing vs Shared everything

Very common terms used in storage industry, some vendors come up with new terms to grab customer attention. NFS is divine, may be some people don't like this statement but NFS still rule the industry. 

Parallel filesystem(storages)

With client dependency it's painful to use but provides better performance in-comparison to NFS based storages.

How frontend (N/S) and backend traffic (E/W) is segregaeted 

frontend traffic is the traffic coming from client nodes. 
backend traffic is the traffic between storage nodes.

## Network throughput calculation

In era of NVMe disk and SCM (storage class memory) disks rarely become bottleneck unless you have very small I/O workload. In our environment usually network is a bottleneck. 

Intel discontinued SCM but afaik they were the first one to introduce this in the market. Other vendors like Samsung and Kioxia still actively work in SCM space. 



