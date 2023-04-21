If you're ready to migrate your services to modernize your infrastructure, you can move your [DFS Namespaces by using Azure Files](/azure/storage/files/files-manage-namespaces?tabs=azure-portal).

## Example of implementation

The below example covers some settings for a DFS-N root consolidation scenario where there’s a requirement of maintaining UNC paths while migrating to Azure. (E.g. \\\oldserver\folder1 must be kept)

![Diagram that shows an example of a DFS Namespaces failover cluster.](../media/DFS-N_cluester_example.png)


1.	Setup 2 servers with DFS-N roles and create in each of them the namespace Stand-alone \\\DFS-A\oldserver# and \\\DFS-B\oldserver# with both targeting the root folders pointing to \\newapplicance\folder1. Keeping the old UNC paths can make it complex depending on the amount of shares.

   Some guidelines can be found in the following link: [Use DFS-N and DFS Root Consolidation with Azure NetApp Files | Microsoft Learn](https://learn.microsoft.com/en-us/azure/azure-netapp-files/use-dfs-n-and-dfs-root-consolidation-with-azure-netapp-files?tabs=windows-gui)

   Make sure you have a DNS entry for the root target.

   Test if both servers can be used to connect to the target folders individually. The next step will provide cluster details.

2.	Setup the Windows Failover Cluster 
The following link has good information for this part: [Deploying DFS Replication on a Windows Failover Cluster](https://techcommunity.microsoft.com/t5/storage-at-microsoft/deploying-dfs-replication-on-a-windows-failover-cluster-amp-8211/ba-p/423913)

At the end of this step, you will have 3 main network components:

DFS-A -> This is the node A of your cluster (E.g. hostname DFS-server-A.contoso.com, ip 10.0.0.4)

DFS-B -> This is the node B of your cluster (E.g. hostname DFS-server-B.contoso.com, ip 10.0.0.5)

Cluster -> This is the cluster service (E.g. hostname DFS-Cluster.contoso.com, ip 10.0.0.6)

Run powershell in one of your DFS nodes “Get-ClusterResource $IPResourceName | Get-ClusterParameter” to make sure you have the cluster up and running.

Now the cluster must be ready to listen on port 59999. This step is needed for setting up the Load Balancer probe. Copy the following PowerShell script:

   ```powershell
   $ClusterNetworkName = "<MyClusterNetworkName>" # The cluster network name. Use Get-ClusterNetwork on Windows Server 2012 or later to find the name.
   $IPResourceName = "<IPResourceName>" # The IP address resource name.
   $ListenerILBIP = "<n.n.n.n>" # The IP address of the internal load balancer. This is the static IP address for the load balancer that you configured in the Azure portal.
   [int]$ListenerProbePort = <nnnnn>
  
   Import-Module FailoverClusters
   Get-ClusterResource $IPResourceName | Set-ClusterParameter -Multiple @{"Address"="$ListenerILBIP";"ProbePort"=$ListenerProbePort;"SubnetMask"="255.255.255.255";"Network"="$ClusterNetworkName";"EnableDhcp"=0}
   ```
More information about the above script can be found in the SQL cluster documentation: [Configure a load balancer & availability group listener (Azure portal)](https://learn.microsoft.com/en-us/azure/azure-sql/virtual-machines/windows/availability-group-load-balancer-portal-configure?view=azuresql)


3.	Setup the Azure Load Balancer

   The FrontEnd IP should be static and have the same IP as the cluster  (E.g. 10.0.0.6)

   The Backend pool should have the 2 nodes (E.g. DFS-A and DFS-B)

   The Probe should check port 59999

   The rule should tied all of them together and have enabled Floating IP

   Final test: Your Load Balancer should be probing successfully the Active DFS node. The passive will fail. Go ahead and failover your cluster to the other node. The Load Balancer should identify that, as you can see in its Metrics.