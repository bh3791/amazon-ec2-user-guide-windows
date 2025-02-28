# EC2 Fleet configuration strategies<a name="ec2-fleet-configuration-strategies"></a>

An *EC2 Fleet* is a group of On\-Demand Instances and Spot Instances\. 

The EC2 Fleet attempts to launch the number of instances that are required to meet the target capacity that you specify in the fleet request\. The fleet can comprise only On\-Demand Instances, only Spot Instances, or a combination of both On\-Demand Instances and Spot Instances\. The request for Spot Instances is fulfilled if there is available capacity and the maximum price per hour for your request exceeds the Spot price\. The fleet also attempts to maintain its target capacity if your Spot Instances are interrupted\.

You can also set a maximum amount per hour that you’re willing to pay for your fleet, and EC2 Fleet launches instances until it reaches the maximum amount\. When the maximum amount you're willing to pay is reached, the fleet stops launching instances even if it hasn’t met the target capacity\.

A *Spot capacity pool* is a set of unused EC2 instances with the same instance type and Availability Zone\. When you create an EC2 Fleet, you can include multiple launch specifications, which vary by instance type, Availability Zone, subnet, and maximum price\. The fleet selects the Spot capacity pools that are used to fulfill the request, based on the launch specifications included in your request, and the configuration of the request\. The Spot Instances come from the selected pools\.

An EC2 Fleet enables you to provision large amounts of EC2 capacity that makes sense for your application based on number of cores or instances, or amount of memory\. For example, you can specify an EC2 Fleet to launch a target capacity of 200 instances, of which 130 are On\-Demand Instances and the rest are Spot Instances\.

Use the appropriate configuration strategies to create an EC2 Fleet that meets your needs\.

**Topics**
+ [Planning an EC2 Fleet](#plan-ec2-fleet)
+ [EC2 Fleet request types](#ec2-fleet-request-type)
+ [Allocation strategies for Spot Instances](#ec2-fleet-allocation-strategy)
+ [Configure EC2 Fleet for On\-Demand backup](#ec2-fleet-on-demand-backup)
+ [Capacity Rebalancing](#ec2-fleet-capacity-rebalance)
+ [Maximum price overrides](#ec2-fleet-price-overrides)
+ [Control spending](#ec2-fleet-control-spending)
+ [EC2 Fleet instance weighting](#ec2-fleet-instance-weighting)
+ [Tutorial: Use EC2 Fleet with instance weighting](#ec2-fleet-instance-weighting-walkthrough)
+ [Tutorial: Use EC2 Fleet with On\-Demand as the primary capacity](#ec2-fleet-on-demand-walkthrough)

## Planning an EC2 Fleet<a name="plan-ec2-fleet"></a>

When planning your EC2 Fleet, we recommend that you do the following:
+ Determine whether you want to create an EC2 Fleet that submits a synchronous or asynchronous one\-time request for the desired target capacity, or one that maintains a target capacity over time\. For more information, see [EC2 Fleet request types](#ec2-fleet-request-type)\.
+ Determine the instance types that meet your application requirements\.
+ If you plan to include Spot Instances in your EC2 Fleet, review [Spot Best Practices](https://aws.amazon.com/ec2/spot/getting-started/#bestpractices) before you create the fleet\. Use these best practices when you plan your fleet so that you can provision the instances at the lowest possible price\.
+ Determine the target capacity for your EC2 Fleet\. You can set target capacity in instances or in custom units\. For more information, see [EC2 Fleet instance weighting](#ec2-fleet-instance-weighting)\.
+ Determine what portion of the EC2 Fleet target capacity must be On\-Demand capacity and Spot capacity\. You can specify 0 for On\-Demand capacity or Spot capacity, or both\.
+ Determine your price per unit, if you are using instance weighting\. To calculate the price per unit, divide the price per instance hour by the number of units \(or weight\) that this instance represents\. If you are not using instance weighting, the default price per unit is the price per instance hour\.
+ Determine the maximum amount per hour that you’re willing to pay for your fleet\. For more information, see [Control spending](#ec2-fleet-control-spending)\.
+ Review the possible options for your EC2 Fleet\. For more information, see the [EC2 Fleet JSON configuration file reference](manage-ec2-fleet.md#ec2-fleet-json-reference)\. For EC2 Fleet configuration examples, see [EC2 Fleet example configurations](ec2-fleet-examples.md)\.

## EC2 Fleet request types<a name="ec2-fleet-request-type"></a>

There are three types of EC2 Fleet requests: 

`instant`  
If you configure the request type as `instant`, EC2 Fleet places a synchronous one\-time request for your desired capacity\. In the API response, it returns the instances that launched, along with errors for those instances that could not be launched\.

`request`  
If you configure the request type as `request`, EC2 Fleet places an asynchronous one\-time request for your desired capacity\. Thereafter, if capacity is diminished because of Spot interruptions, the fleet does not attempt to replenish Spot Instances, nor does it submit requests in alternative Spot capacity pools if capacity is unavailable\.

`maintain`  
\(Default\) If you configure the request type as `maintain`, EC2 Fleet places an asynchronous request for your desired capacity, and maintains capacity by automatically replenishing any interrupted Spot Instances\.

All three types of requests benefit from an allocation strategy\. For more information, see [Allocation strategies for Spot Instances](#ec2-fleet-allocation-strategy)\.

## Allocation strategies for Spot Instances<a name="ec2-fleet-allocation-strategy"></a>

The allocation strategy for your EC2 Fleet determines how it fulfills your request for Spot Instances from the possible Spot capacity pools represented by its launch specifications\. The following are the allocation strategies that you can specify in your fleet:

`lowest-price`  
The Spot Instances come from the Spot capacity pool with the lowest price\. This is the default strategy\.

`diversified`  
The Spot Instances are distributed across all Spot capacity pools\.

`capacity-optimized`  
The Spot Instances come from the Spot capacity pool with optimal capacity for the number of instances that are launching\. You can optionally set a priority for each instance type in your fleet using `capacity-optimized-prioritized`\. EC2 Fleet optimizes for capacity first, but honors instance type priorities on a best\-effort basis\.  
With Spot Instances, pricing changes slowly over time based on long\-term trends in supply and demand, but capacity fluctuates in real time\. The `capacity-optimized` strategy automatically launches Spot Instances into the most available pools by looking at real\-time capacity data and predicting which are the most available\. This works well for workloads such as big data and analytics, image and media rendering, machine learning, and high performance computing that may have a higher cost of interruption associated with restarting work and checkpointing\. By offering the possibility of fewer interruptions, the `capacity-optimized` strategy can lower the overall cost of your workload\.  
Alternatively, you can use the `capacity-optimized-prioritized` allocation strategy with a priority parameter to order instance types from highest to lowest priority\. You can set the same priority for different instance types\. EC2 Fleet will optimize for capacity first, but will honor instance type priorities on a best\-effort basis \(for example, if honoring the priorities will not significantly affect EC2 Fleet's ability to provision optimal capacity\)\. This is a good option for workloads where the possibility of disruption must be minimized and the preference for certain instance types matters\. Using priorities is supported only if your fleet uses a launch template\. Note that when you set the priority for `capacity-optimized-prioritized`, the same priority is also applied to your On\-Demand Instances if the On\-Demand `AllocationStrategy` is set to `prioritized`\.

`InstancePoolsToUseCount`  
The Spot Instances are distributed across the number of Spot capacity pools that you specify\. This parameter is valid only when used in combination with `lowest-price`\.

### Maintaining target capacity<a name="ec2-fleet-maintain-fleet-capacity"></a>

After Spot Instances are terminated due to a change in the Spot price or available capacity of a Spot capacity pool, an EC2 Fleet of type `maintain` launches replacement Spot Instances\. If the allocation strategy is `lowest-price`, the fleet launches replacement instances in the pool where the Spot price is currently the lowest\. If the allocation strategy is `lowest-price` in combination with `InstancePoolsToUseCount`, the fleet selects the Spot capacity pools with the lowest price and launches Spot Instances across the number of Spot capacity pools that you specify\. If the allocation strategy is `capacity-optimized`, the fleet launches replacement instances in the pool that has the most available Spot Instance capacity\. If the allocation strategy is `diversified`, the fleet distributes the replacement Spot Instances across the remaining pools\.

### Choose the appropriate allocation strategy<a name="ec2-fleet-allocation-use-cases"></a>

You can optimize your fleet based on your use case\.

If your fleet runs workloads that may have a higher cost of interruption associated with restarting work and checkpointing, then use the `capacity-optimized` strategy\. This strategy offers the possibility of fewer interruptions, which can lower the overall cost of your workload\. Use the `capacity-optimized-prioritized` strategy for workloads where the possibility of disruption must be minimized and the preference for certain instance types matters\.

If your fleet is small or runs for a short time, the probability that your Spot Instances will be interrupted is low, even with all of the instances in a single Spot capacity pool\. Therefore, the `lowest-price` strategy is likely to meet your needs while providing the lowest cost\.

If your fleet is large or runs for a long time, you can improve the availability of your fleet by distributing the Spot Instances across multiple pools using the `diversified` strategy\. For example, if your EC2 Fleet specifies 10 pools and a target capacity of 100 instances, the fleet launches 10 Spot Instances in each pool\. If the Spot price for one pool exceeds your maximum price for this pool, only 10% of your fleet is affected\. Using this strategy also makes your fleet less sensitive to increases in the Spot price in any one pool over time\. With the `diversified` strategy, the EC2 Fleet does not launch Spot Instances into any pools with a Spot price that is equal to or higher than the [On\-Demand price](https://aws.amazon.com/ec2/pricing/)\.

To create a cheap and diversified fleet, use the `lowest-price` strategy in combination with `InstancePoolsToUseCount`\. You can use a low or high number of Spot capacity pools across which to allocate your Spot Instances\. For example, if you run batch processing, we recommend specifying a low number of Spot capacity pools \(for example, `InstancePoolsToUseCount=2`\) to ensure that your queue always has compute capacity while maximizing savings\. If you run a web service, we recommend specifying a high number of Spot capacity pools \(for example, `InstancePoolsToUseCount=10`\) to minimize the impact if a Spot capacity pool becomes temporarily unavailable\.

### Configure EC2 Fleet for cost optimization<a name="ec2-fleet-strategy-cost-optimization"></a>

To optimize the costs for your use of Spot Instances, specify the `lowest-price` allocation strategy so that EC2 Fleet automatically deploys the least expensive combination of instance types and Availability Zones based on the current Spot price\.

For On\-Demand Instance target capacity, EC2 Fleet always selects the cheapest instance type based on the public On\-Demand price, while continuing to follow the allocation strategy \(either `lowest-price`, `capacity-optimized`, or `diversified`\) for Spot Instances\.

### Configure EC2 Fleet for cost optimization and diversification<a name="ec2-fleet-strategy-cost-optimization-and-diversified"></a>

To create a fleet of Spot Instances that is both cheap and diversified, use the `lowest-price` allocation strategy in combination with `InstancePoolsToUseCount`\. EC2 Fleet automatically deploys the least expensive combination of instance types and Availability Zones based on the current Spot price across the number of Spot capacity pools that you specify\. This combination can be used to avoid the most expensive Spot Instances\.

### Configure EC2 Fleet for capacity optimization<a name="ec2-fleet-strategy-capacity-optimized"></a>

To launch Spot Instances into the most\-available Spot capacity pools, use the `capacity-optimized` allocation strategy\. For an example configuration, see [Example 10: Launch Spot Instances in a capacity\-optimized fleet](ec2-fleet-examples.md#ec2-fleet-config10)\.

You can also express your pool priorities by using the `capacity-optimized-prioritized` allocation strategy and then setting the order of instance types to use from highest to lowest priority\. Using priorities is supported only if your fleet uses a launch template\. Note that when you set priorities for `capacity-optimized-prioritized`, the same priorities are also applied to your On\-Demand Instances if the On\-Demand `AllocationStrategy` is set to `prioritized`\. For an example configuration, see [Example 11: Launch Spot Instances in a capacity\-optimized fleet with priorities](ec2-fleet-examples.md#ec2-fleet-config11)\.

## Configure EC2 Fleet for On\-Demand backup<a name="ec2-fleet-on-demand-backup"></a>

If you have urgent, unpredictable scaling needs, such as a news website that must scale during a major news event or game launch, we recommend that you specify alternative instance types for your On\-Demand Instances, in the event that your preferred option does not have sufficient available capacity\. For example, you might prefer `c5.2xlarge` On\-Demand Instances, but if there is insufficient available capacity, you'd be willing to use some `c4.2xlarge` instances during peak load\. In this case, EC2 Fleet attempts to fulfill all of your target capacity using `c5.2xlarge` instances, but if there is insufficient capacity, it automatically launches `c4.2xlarge` instances to fulfill the target capacity\.

### Prioritize instance types for On\-Demand capacity<a name="ec2-fleet-on-demand-priority"></a>

When EC2 Fleet attempts to fulfill your On\-Demand capacity, it defaults to launching the lowest\-priced instance type first\. If `AllocationStrategy` is set to `prioritized`, EC2 Fleet uses priority to determine which instance type to use first in fulfilling On\-Demand capacity\. The priority is assigned to the launch template override, and the highest priority is launched first\. 

For example, you have configured three launch template overrides, each with a different instance type: `c3.large`, `c4.large`, and `c5.large`\. The On\-Demand price for `c5.large` is less than the price for `c4.large`\. `c3.large` is the cheapest\. If you do not use priority to determine the order, the fleet fulfills On\-Demand capacity by starting with `c3.large`, and then `c5.large`\. Because you often have unused Reserved Instances for `c4.large`, you can set the launch template override priority so that the order is `c4.large`, `c3.large`, and then `c5.large`\.

### Use Capacity Reservations for On\-Demand Instances<a name="ec2-fleet-on-demand-capacity-reservations"></a>

You can configure a fleet to use On\-Demand Capacity Reservations first when launching On\-Demand Instances by setting the usage strategy for Capacity Reservations to `use-capacity-reservations-first`\. You can use this setting in conjunction with the allocation strategy for On\-Demand Instances \(`lowest-price` or `prioritized`\)\.

When unused Capacity Reservations are used to fulfil On\-Demand capacity:
+ The fleet uses unused Capacity Reservations to fulfill On\-Demand capacity up to the target On\-Demand capacity\.
+ If multiple instance pools have unused Capacity Reservations, the On\-Demand allocation strategy \(`lowest-price` or `prioritized`\) is applied\.
+ If the number of unused Capacity Reservations is less than the On\-Demand target capacity, the remaining On\-Demand target capacity is launched according to the On\-Demand allocation strategy \(`lowest-price` or `prioritized`\)\.

You can only use unused On\-Demand Capacity Reservations for fleets of type `instant`\.

For examples of how to configure a fleet to use Capacity Reservations to fulfil On\-Demand capacity, see [EC2 Fleet example configurations](ec2-fleet-examples.md)\. For more information, see [On\-Demand Capacity Reservations](ec2-capacity-reservations.md) and the [On\-Demand Capacity Reservation FAQs](http://aws.amazon.com/ec2/faqs/#On-Demand_Capacity_Reservation)\.

## Capacity Rebalancing<a name="ec2-fleet-capacity-rebalance"></a>

You can configure EC2 Fleet to launch a replacement Spot Instance when Amazon EC2 emits a rebalance recommendation to notify you that a Spot Instance is at an elevated risk of interruption\. Capacity Rebalancing helps you maintain workload availability by proactively augmenting your fleet with a new Spot Instance before a running instance is interrupted by Amazon EC2\. For more information, see [EC2 instance rebalance recommendations](rebalance-recommendations.md)\.

To configure EC2 Fleet to launch a replacement Spot Instance, use the [create\-fleet](https://docs.aws.amazon.com/cli/latest/reference/ec2/create-fleet.html) \(AWS CLI\) command and the relevant parameters in the `MaintenanceStrategies` structure\. For more information, see the [example launch configuration](ec2-fleet-examples.md#ec2-fleet-config9)\.

**Limitations**
+ Only available for fleets of type `maintain`\.
+ When the fleet is running, you can't modify the Capacity Rebalancing setting\. To change the Capacity Rebalancing setting, you must delete the fleet and create a new fleet\.

**Considerations**

If you configure an EC2 Fleet for Capacity Rebalancing, consider the following:

**EC2 Fleet can launch new replacement Spot Instances until fulfilled capacity is double target capacity**  
When an EC2 Fleet is configured for Capacity Rebalancing, the fleet attempts to launch a new replacement Spot Instance for every Spot Instance that receives a rebalance recommendation\. After a Spot Instance receives a rebalance recommendation, it is no longer counted as part of the fulfilled capacity, and EC2 Fleet does not automatically terminate the instance\. This gives you the opportunity to perform [rebalancing actions](rebalance-recommendations.md#rebalancing-actions) on the instance\. Thereafter, you can terminate the instance, or you can leave it running\.  
If your fleet reaches double its target capacity, it stops launching new replacement instances even if the replacement instances themselves receive a rebalance recommendation\.  
For example, you create an EC2 Fleet with a target capacity of 100 Spot Instances\. All the Spot Instances receive a rebalance recommendation, which causes EC2 Fleet to launch 100 replacement Spot Instances\. This raises the number of fulfilled Spot Instances to 200, which is double the target capacity\. Some of the replacement instances receive a rebalance recommendation, but no more replacement instances are launched because the fleet cannot exceed double its target capacity\.   
Note that you are charged for all of the instances while they are running\.

**We recommend that you manually terminate Spot Instances that receive a rebalance recommendation**  
If you configure your EC2 Fleet for Capacity Rebalancing, we recommend that you monitor the rebalance recommendation signal that is received by the Spot Instances in the fleet\. By monitoring the signal, you can quickly perform [rebalancing actions](rebalance-recommendations.md#rebalancing-actions) on the affected instances before Amazon EC2 interrupts them, and then you can manually terminate them\. If you do not terminate the instances, you continue paying for them while they are running\. EC2 Fleet does not automatically terminate the instances that receive a rebalance recommendation\.  
You can set up notifications using Amazon EventBridge or instance metadata\. For more information, see [Monitor rebalance recommendation signals](rebalance-recommendations.md#monitor-rebalance-recommendations)\.

**EC2 Fleet does not count instances that receive a rebalance recommendation when calculating fulfilled capacity during scale in or out**  
If your EC2 Fleet is configured for Capacity Rebalancing, and you change the target capacity to either scale in or scale out, the fleet does not count the instances that are marked for rebalance as part of the fulfilled capacity, as follows:  
+ Scale in – If you decrease your desired target capacity, the fleet terminates instances that are not marked for rebalance until the desired capacity is reached\. The instances that are marked for rebalance are not counted towards the fulfilled capacity\.

  For example, you create an EC2 Fleet with a target capacity of 100 Spot Instances\. 10 instances receive a rebalance recommendation, so the fleet launches 10 new replacement instances, resulting in a fulfilled capacity of 110 instances\. You then reduce the target capacity to 50 \(scale in\), but the fulfilled capacity is actually 60 instances because the 10 instances that are marked for rebalance are not terminated by the fleet\. You need to manually terminate these instances, or you can leave them running\.
+ Scale out – If you increase your desired target capacity, the fleet launches new instances until the desired capacity is reached\. The instances that are marked for rebalance are not counted towards the fulfilled capacity\. 

  For example, you create an EC2 Fleet with a target capacity of 100 Spot Instances\. 10 instances receive a rebalance recommendation, so the fleet launches 10 new replacement instances, resulting in a fulfilled capacity of 110 instances\. You then increase the target capacity to 200 \(scale out\), but the fulfilled capacity is actually 210 instances because the 10 instances that are marked for rebalance are not counted by the fleet as part of the target capacity\. You need to manually terminate these instances, or you can leave them running\.

**Provide as many Spot capacity pools in the request as possible**  
Configure your EC2 Fleet to use multiple instance types and Availability Zones\. This provides the flexibility to launch Spot Instances in various Spot capacity pools\. For more information, see [Be flexible about instance types and Availability Zones](spot-best-practices.md#be-instance-type-flexible)\.

**Configure your EC2 Fleet to use the most optimal Spot capacity pools**  
Use the `capacity-optimized` allocation strategy to ensure that replacement Spot Instances are launched in the most optimal Spot capacity pools\. For more information, see [Use the capacity optimized allocation strategy](spot-best-practices.md#use-capacity-optimized-allocation-strategy)\. 

## Maximum price overrides<a name="ec2-fleet-price-overrides"></a>

Each EC2 Fleet can either include a global maximum price, or use the default \(the On\-Demand price\)\. The fleet uses this as the default maximum price for each of its launch specifications\.

You can optionally specify a maximum price in one or more launch specifications\. This price is specific to the launch specification\. If a launch specification includes a specific price, the EC2 Fleet uses this maximum price, overriding the global maximum price\. Any other launch specifications that do not include a specific maximum price still use the global maximum price\.

## Control spending<a name="ec2-fleet-control-spending"></a>

EC2 Fleet stops launching instances when it has met one of the following parameters: the `TotalTargetCapacity` or the `MaxTotalPrice` \(the maximum amount you’re willing to pay\)\. To control the amount you pay per hour for your fleet, you can specify the `MaxTotalPrice`\. When the maximum total price is reached, EC2 Fleet stops launching instances even if it hasn’t met the target capacity\.

The following examples show two different scenarios\. In the first, EC2 Fleet stops launching instances when it has met the target capacity\. In the second, EC2 Fleet stops launching instances when it has reached the maximum amount you’re willing to pay \(`MaxTotalPrice`\)\.

**Example: Stop launching instances when target capacity is reached**

Given a request for `m4.large` On\-Demand Instances, where:
+ On\-Demand Price: $0\.10 per hour
+ `OnDemandTargetCapacity`: 10
+ `MaxTotalPrice`: $1\.50

EC2 Fleet launches 10 On\-Demand Instances because the total of $1\.00 \(10 instances x $0\.10\) does not exceed the `MaxTotalPrice` of $1\.50 for On\-Demand Instances\.

**Example: Stop launching instances when maximum total price is reached**

Given a request for `m4.large` On\-Demand Instances, where:
+ On\-Demand Price: $0\.10 per hour
+ `OnDemandTargetCapacity`: 10
+ `MaxTotalPrice`: $0\.80

If EC2 Fleet launches the On\-Demand target capacity \(10 On\-Demand Instances\), the total cost per hour would be $1\.00\. This is more than the amount \($0\.80\) specified for `MaxTotalPrice` for On\-Demand Instances\. To prevent spending more than you're willing to pay, EC2 Fleet launches only 8 On\-Demand Instances \(below the On\-Demand target capacity\) because launching more would exceed the `MaxTotalPrice` for On\-Demand Instances\.

## EC2 Fleet instance weighting<a name="ec2-fleet-instance-weighting"></a>

When you create an EC2 Fleet, you can define the capacity units that each instance type would contribute to your application's performance\. You can then adjust your maximum price for each launch specification by using *instance weighting*\.

By default, the price that you specify is *per instance hour*\. When you use the instance weighting feature, the price that you specify is *per unit hour*\. You can calculate your price per unit hour by dividing your price for an instance type by the number of units that it represents\. EC2 Fleet calculates the number of instances to launch by dividing the target capacity by the instance weight\. If the result isn't an integer, the fleet rounds it up to the next integer, so that the size of your fleet is not below its target capacity\. The fleet can select any pool that you specify in your launch specification, even if the capacity of the instances launched exceeds the requested target capacity\.

The following table includes examples of calculations to determine the price per unit for an EC2 Fleet with a target capacity of 10\.


| Instance type | Instance weight | Target capacity | Number of instances launched | Price per instance hour | Price per unit hour | 
| --- | --- | --- | --- | --- | --- | 
| r3\.xlarge |  2  | 10 |  5 \(10 divided by 2\)  |  $0\.05  |  $0\.025 \(\.05 divided by 2\)  | 
| r3\.8xlarge |  8  | 10 |  2 \(10 divided by 8, result rounded up\)  |  $0\.10  |  $0\.0125 \(\.10 divided by 8\)  | 

Use EC2 Fleet instance weighting as follows to provision the target capacity that you want in the pools with the lowest price per unit at the time of fulfillment:

1. Set the target capacity for your EC2 Fleet either in instances \(the default\) or in the units of your choice, such as virtual CPUs, memory, storage, or throughput\.

1. Set the price per unit\.

1. For each launch specification, specify the weight, which is the number of units that the instance type represents toward the target capacity\.

**Instance weighting example**  
Consider an EC2 Fleet request with the following configuration:
+ A target capacity of 24
+ A launch specification with an instance type `r3.2xlarge` and a weight of 6
+ A launch specification with an instance type `c3.xlarge` and a weight of 5

The weights represent the number of units that instance type represents toward the target capacity\. If the first launch specification provides the lowest price per unit \(price for `r3.2xlarge` per instance hour divided by 6\), the EC2 Fleet would launch four of these instances \(24 divided by 6\)\.

If the second launch specification provides the lowest price per unit \(price for `c3.xlarge` per instance hour divided by 5\), the EC2 Fleet would launch five of these instances \(24 divided by 5, result rounded up\)\.

**Instance weighting and allocation strategy**  
Consider an EC2 Fleet request with the following configuration:
+ A target capacity of 30 Spot Instances
+ A launch specification with an instance type `c3.2xlarge` and a weight of 8
+ A launch specification with an instance type `m3.xlarge` and a weight of 8
+ A launch specification with an instance type `r3.xlarge` and a weight of 8

The EC2 Fleet would launch four instances \(30 divided by 8, result rounded up\)\. With the `lowest-price` strategy, all four instances come from the pool that provides the lowest price per unit\. With the `diversified` strategy, the fleet launches one instance in each of the three pools, and the fourth instance in whichever of the three pools provides the lowest price per unit\.

## Tutorial: Use EC2 Fleet with instance weighting<a name="ec2-fleet-instance-weighting-walkthrough"></a>

This tutorial uses a fictitious company called Example Corp to illustrate the process of requesting an EC2 Fleet using instance weighting\.

### Objective<a name="ec2-fleet-instance-weighting-walkthrough-objective"></a>

Example Corp, a pharmaceutical company, wants to use the computational power of Amazon EC2 for screening chemical compounds that might be used to fight cancer\.

### Planning<a name="ec2-fleet-instance-weighting-walkthrough-planning"></a>

Example Corp first reviews [Spot Best Practices](https://aws.amazon.com/ec2/spot/getting-started/#bestpractices)\. Next, Example Corp determines the requirements for their EC2 Fleet\.

**Instance types**  
Example Corp has a compute\- and memory\-intensive application that performs best with at least 60 GB of memory and eight virtual CPUs \(vCPUs\)\. They want to maximize these resources for the application at the lowest possible price\. Example Corp decides that any of the following EC2 instance types would meet their needs:


| Instance type | Memory \(GiB\) | vCPUs | 
| --- | --- | --- | 
|  r3\.2xlarge  |  61  |  8  | 
|  r3\.4xlarge  |  122  |  16  | 
|  r3\.8xlarge  |  244  |  32  | 

**Target capacity in units**  
With instance weighting, target capacity can equal a number of instances \(the default\) or a combination of factors such as cores \(vCPUs\), memory \(GiBs\), and storage \(GBs\)\. By considering the base for their application \(60 GB of RAM and eight vCPUs\) as one unit, Example Corp decides that 20 times this amount would meet their needs\. So the company sets the target capacity of their EC2 Fleet request to 20\.

**Instance weights**  
After determining the target capacity, Example Corp calculates instance weights\. To calculate the instance weight for each instance type, they determine the units of each instance type that are required to reach the target capacity as follows:
+ r3\.2xlarge \(61\.0 GB, 8 vCPUs\) = 1 unit of 20
+ r3\.4xlarge \(122\.0 GB, 16 vCPUs\) = 2 units of 20
+ r3\.8xlarge \(244\.0 GB, 32 vCPUs\) = 4 units of 20

Therefore, Example Corp assigns instance weights of 1, 2, and 4 to the respective launch configurations in their EC2 Fleet request\.

**Price per unit hour**  
Example Corp uses the [On\-Demand price](https://aws.amazon.com/ec2/pricing/) per instance hour as a starting point for their price\. They could also use recent Spot prices, or a combination of the two\. To calculate the price per unit hour, they divide their starting price per instance hour by the weight\. For example:


| Instance type | On\-Demand price | Instance weight | Price per unit hour | 
| --- | --- | --- | --- | 
|  r3\.2xLarge  |  $0\.7  |  1  |  $0\.7  | 
|  r3\.4xLarge  |  $1\.4  |  2  |  $0\.7  | 
|  r3\.8xLarge  |  $2\.8  |  4  |  $0\.7  | 

Example Corp could use a global price per unit hour of $0\.7 and be competitive for all three instance types\. They could also use a global price per unit hour of $0\.7 and a specific price per unit hour of $0\.9 in the `r3.8xlarge` launch specification\.

### Verify permissions<a name="ec2-fleet-instance-weighting-walkthrough-permissions"></a>

Before creating an EC2 Fleet, Example Corp verifies that it has an IAM role with the required permissions\. For more information, see [EC2 Fleet prerequisites](manage-ec2-fleet.md#ec2-fleet-prerequisites)\.

### Create a launch template<a name="ec2-fleet-instance-weighting-create-launch-template"></a>

Next, Example Corp creates a launch template\. The launch template ID is used in the following step\. For more information, see [Create a launch template](ec2-launch-templates.md#create-launch-template)\.

### Create the EC2 Fleet<a name="ec2-fleet-instance-weighting-walkthrough-request"></a>

Example Corp creates a file, `config.json`, with the following configuration for its EC2 Fleet\. In the following example, replace the resource identifiers with your own resource identifiers\.

```
{ 
    "LaunchTemplateConfigs": [
        {
            "LaunchTemplateSpecification": {
                "LaunchTemplateId": "lt-07b3bc7625cdab851", 
                "Version": "1"
            }, 
            "Overrides": [
                {
                    "InstanceType": "r3.2xlarge", 
                    "SubnetId": "subnet-482e4972", 
                    "WeightedCapacity": 1
                },
                {
                    "InstanceType": "r3.4xlarge", 
                    "SubnetId": "subnet-482e4972", 
                    "WeightedCapacity": 2
                },
                {
                    "InstanceType": "r3.8xlarge", 
                    "MaxPrice": "0.90", 
                    "SubnetId": "subnet-482e4972", 
                    "WeightedCapacity": 4
                }
            ]
        }
    ], 
    "TargetCapacitySpecification": {
        "TotalTargetCapacity": 20, 
        "DefaultTargetCapacityType": "spot"
    }
}
```

Example Corp creates the EC2 Fleet using the following [create\-fleet](https://docs.aws.amazon.com/cli/latest/reference/ec2/create-fleet.html) command\.

```
aws ec2 create-fleet \
    --cli-input-json file://config.json
```

For more information, see [Create an EC2 Fleet](manage-ec2-fleet.md#create-ec2-fleet)\.

### Fulfillment<a name="ec2-fleet-instance-weighting-walkthrough-fulfillment"></a>

The allocation strategy determines which Spot capacity pools your Spot Instances come from\.

With the `lowest-price` strategy \(which is the default strategy\), the Spot Instances come from the pool with the lowest price per unit at the time of fulfillment\. To provide 20 units of capacity, the EC2 Fleet launches either 20 `r3.2xlarge` instances \(20 divided by 1\), 10 `r3.4xlarge` instances \(20 divided by 2\), or 5 `r3.8xlarge` instances \(20 divided by 4\)\.

If Example Corp used the `diversified` strategy, the Spot Instances would come from all three pools\. The EC2 Fleet would launch 6 `r3.2xlarge` instances \(which provide 6 units\), 3 `r3.4xlarge` instances \(which provide 6 units\), and 2 `r3.8xlarge` instances \(which provide 8 units\), for a total of 20 units\.

## Tutorial: Use EC2 Fleet with On\-Demand as the primary capacity<a name="ec2-fleet-on-demand-walkthrough"></a>

This tutorial uses a fictitious company called ABC Online to illustrate the process of requesting an EC2 Fleet with On\-Demand as the primary capacity, and Spot capacity if available\.

### Objective<a name="ec2-fleet-on-demand-walkthrough-objective"></a>

ABC Online, a restaurant delivery company, wants to be able to provision Amazon EC2 capacity across EC2 instance types and purchasing options to achieve their desired scale, performance, and cost\.

### Plan<a name="ec2-fleet-on-demand-walkthrough-planning"></a>

ABC Online requires a fixed capacity to operate during peak periods, but would like to benefit from increased capacity at a lower price\. ABC Online determines the following requirements for their EC2 Fleet:
+ On\-Demand Instance capacity – ABC Online requires 15 On\-Demand Instances to ensure that they can accommodate traffic at peak periods\.
+ Spot Instance capacity – ABC Online would like to improve performance, but at a lower price, by provisioning 5 Spot Instances\.

### Verify permissions<a name="ec2-fleet-on-demand-walkthrough-permissions"></a>

Before creating an EC2 Fleet, ABC Online verifies that it has an IAM role with the required permissions\. For more information, see [EC2 Fleet prerequisites](manage-ec2-fleet.md#ec2-fleet-prerequisites)\.

### Create a launch template<a name="ec2-fleet-on-demand-walkthrough-create-launch-template"></a>

Next, ABC Online creates a launch template\. The launch template ID is used in the following step\. For more information, see [Create a launch template](ec2-launch-templates.md#create-launch-template)\.

### Create the EC2 Fleet<a name="ec2-fleet-on-demand-walkthrough-request"></a>

ABC Online creates a file, `config.json`, with the following configuration for its EC2 Fleet\. In the following example, replace the resource identifiers with your own resource identifiers\.

```
{
    "LaunchTemplateConfigs": [
        {
            "LaunchTemplateSpecification": {
                "LaunchTemplateId": "lt-07b3bc7625cdab851",
                "Version": "2"
            }

        }
    ],
    "TargetCapacitySpecification": {
        "TotalTargetCapacity": 20,
        "OnDemandTargetCapacity":15,
        "DefaultTargetCapacityType": "spot"
    }
}
```

ABC Online creates the EC2 Fleet using the following [create\-fleet](https://docs.aws.amazon.com/cli/latest/reference/ec2/create-fleet.html) command\.

```
aws ec2 create-fleet \
    --cli-input-json file://config.json
```

For more information, see [Create an EC2 Fleet](manage-ec2-fleet.md#create-ec2-fleet)\.

### Fulfillment<a name="ec2-fleet-on-demand-walkthrough-fulfillment"></a>

The allocation strategy determines that the On\-Demand capacity is always fulfilled, while the balance of the target capacity is fulfilled as Spot if there is capacity and availability\.