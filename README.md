# SCE GPU Operation Playbook

## Installation
The installation of the GPU cluster utilizes the OCI stack from the provided link: [oracle-quickstart/oci-hpc](https://github.com/oracle-quickstart/oci-hpc)

![01_Stack_to_Create_Slurm_based_GPU_Cluster](./images/01_Installation/01_Stack.png)

When configuring HPC + GPU clustering in OCI, the official recommendation is to use OCI-validated images. OCI performs testing and validation across OS type, compute shape, OFED, and CUDA versions, and publishes the validated images on the Confluence page as shown below. These images are updated twice a year.
[GPU Images (Feb/Mar 2026) Release - Compute - ALM Confluence](https://confluence.oraclecorp.com/confluence/pages/viewpage.action?pageId=18835757732)

![GPU Images By OS Type & Shape](./images/01_Installation/02_GPU_Images.png)


## Operation

### Cluster Network Resizing (via resize.sh)

Cluster resizing refers to ability to add or remove nodes from an existing cluster network. Apart from add/remove, the resize.py script can also be used to reconfigure the nodes.

Resizing of HPC cluster with Cluster Network consist of 2 major sub-steps:

Add/Remove node (IaaS provisioning) to cluster – uses OCI Python SDK

Configure the nodes (uses Ansible)

Configures newly added nodes to be ready to run the jobs
Reconfigure services like Slurm to recognize new nodes on all nodes
Update rest of the nodes, when any node/s are removed (eg: Slurm config, /etc/hosts, etc.)

```
/opt/oci-hpc/bin/resize.sh -h
usage: resize.sh [-h] [--compartment_ocid COMPARTMENT_OCID]
                 [--cluster_name CLUSTER_NAME] [--nodes NODES [NODES ...]]
                 [--no_reconfigure] [--user_logging] [--force] [--remove_unreachable]
                 [{add,remove,remove_unreachable,list,reconfigure}] [number] [--quiet]
Script to resize the CN

positional arguments:
  {add,remove,remove_unreachable,list,reconfigure}
                              Mode type. add/remove node options, implicitly
                              configures newly added nodes. Also implicitly
                              reconfigure/restart services like Slurm to recognize
                              new nodes. Similarly for remove option, terminates
                              nodes and implicitly reconfigure/restart services like
                              Slurm on rest of the cluster nodes to remove reference
                              to deleted nodes. IMPORTANT: remove or remove_unreachable 
                              means delete the node from the cluster which means terminate 
                              the node. remove_unreachable should be used to remove specific 
                              nodes which are no longer reachable via ssh. It gives you control 
                              on which nodes will be terminated by passing the --nodes parameter.
number                        Number of nodes to add or delete if a list of
                              hostnames is not defined.

optional arguments:
  -h, --help            show this help message and exit
  --compartment_ocid COMPARTMENT_OCID
                        OCID of the compartment, defaults to the Compartment
                        OCID of the localhost
  --cluster_name CLUSTER_NAME
                        Name of the cluster to resize. Defaults to the name
                        included in the controller
  --nodes NODES [NODES ...]
                        List of nodes to delete
  --no_reconfigure      If present. Does not rerun the playbooks
  --user_logging        If present. Use the default settings in ~/.oci/config
                        to connect to the API. Default is using
                        instance_principal
  --force               If present. Nodes will be removed even if the destroy
                        playbook failed
  --ansible_crucial     If present during reconfiguration, only crucial
                        ansible playbooks will be executed on the live nodes.
                        Non live nodes will be removed
  --remove_unreachable  If present, ALL nodes that are not sshable will be terminated 
                        before running the action that was requested (Example Adding a node). 
                        CAUTION: Use this only if you want to remove ALL nodes that 
                        are unreachable. Instead, remove specific nodes that are 
                        unreachable by using positional argument remove_unreachable. 
  --quiet               If present, the script will not prompt for a response when 
                        removing nodes and will not give a reminder to save data 
                        from nodes that are being removed
```

***Add nodes*** 

Consist of the following sub-steps:
- Add node (IaaS provisioning) to cluster – uses OCI Python SDK 
- Configure the nodes (uses Ansible)
  -  Configures newly added nodes to be ready to run the jobs
  -  Reconfigure services like Slurm to recognize new nodes on all nodes

Add one node 
```
/opt/oci-hpc/bin/resize.sh add 1

```

Add three nodes to cluster compute-1-hpc
```
/opt/oci-hpc/bin/resize.sh add 3 --cluster_name compute-1-hpc

```


***Remove nodes*** 

Consist of the following sub-steps:
- Remove node/s (IaaS termination) from cluster – uses OCI Python SDK 
- Reconfigure rest of the nodes in the cluster  (uses Ansible)
  -  Remove reference to removed node/s on rest of the nodes (eg: update /etc/hosts, slurm configs, etc.)
 

Remove specific node:  
```
/opt/oci-hpc/bin/resize.sh remove --nodes inst-dpi8e-assuring-woodcock
```
or 

Remove a list of nodes (space seperated):  
```
/opt/oci-hpc/bin/resize.sh remove --nodes inst-dpi8e-assuring-woodcock inst-ed5yh-assuring-woodcock
```
or

Remove one node randomly:  
```
/opt/oci-hpc/bin/resize.sh remove 1
```
or 

Remove 3 nodes randomly from compute-1-hpc:  
```
/opt/oci-hpc/bin/resize.sh remove 3 --cluster_name compute-1-hpc

```
or 

Remove 3 nodes randomly from compute-1-hpc but do not prompt for a response when removing the nodes and do not give a reminder to save data 
from nodes that are being removed :  
```
/opt/oci-hpc/bin/resize.sh remove 3 --cluster_name compute-1-hpc --quiet

```

***Reconfigure nodes*** 

This allows users to reconfigure nodes (Ansible tasks) of the cluster.  

Full reconfiguration of all nodes of the cluster.   This will run the same steps, which are ran when a new cluster is created.   If you manually updated configs which are created/updated as part of cluster configuration, then this command will overwrite your manual changes.   

```
/opt/oci-hpc/bin/resize.sh reconfigure
```

### Automation

***Automation of adding nodes***

You can use the following script to automate the process of adding nodes.
You can create a shell script and schedule it to run at regular intervals with crontab.

Below script checks the number of available nodes in the capacity topology and adds the nodes to the cluster.

```
count=$(/home/ubuntu/bin/oci compute capacity-topology bare-metal-host list \
  --capacity-topology-id <TopologyPolicy OCID> \
  | jq '[.data.items[] | select(."instance-id" == null and  .["instance-shape"] == "<Instance Shape>" and ."lifecycle-details" == "AVAILABLE")] | length')

if [ "$count" -gt 0 ]; then
      /opt/oci-hpc/bin/resize.sh add $count
fi
```

You can check the TopologyPolicy OCID and Instance Shape in the OCI Console or you need to check with your OCI Admin.

![topology_policy](./images/02_Operation/01_topology_policy.png)
![shape](./images/02_Operation/02_shape.png)

## Monitoring

### Horizon Dashboard



## Troubleshooting

### Compute Node GPU Failure

***nvidia-smi***

You can login to the compute node to check detailed info in order to identity the node status.
If you want to check GPU status, you can check with below command.

```
nvidia-smi
```

Below screenshot shows the output of the command. If you see the output like this, it means the GPU is working fine.

![nvidia-smi](images/04_Troubleshooting/01_nvidia-smi.png)

if you see error message, you need to check Slurm cluster node & partition status and the GPU status with below command.

***Slurm status***

```
sinfo
```
Normal output of the command is similar to the following:

![sinfo](images/04_Troubleshooting/02_sinfo.png)

If you see there is drain state, it means the node is not available for job scheduling. You need to talk with your customer / job owner /  cluster admin to resolve this issue.

***GPU / RDMA / PCIe status***

```
sudo python3 /opt/oci-hpc/healthchecks/check_gpu_setup.py
```
Normal output of the command is similar to the following:

![check_gpu_setup](images/04_Troubleshooting/03_check_gpu_setup.png)

if you see error message, it means the GPU is not working fine. You need to talk with your customer to remove this node from the cluster network.

If customer environment uses 3rd party solution for storage, you need to ask customer to remove the node from the storage cluster as well.

***Tagging 'Unhealthy'***

When the issue is caused from GPU failure, you need to tag the node as unhealthy to send this node to repair process.

```
Tag Namespace: ComputeInstanceHostActions
Tag Name: CustomerReportedHostStatus
Tag Values: unhealthy 
```

![unhealthy_tag](images/04_Troubleshooting/04_unhealthy_tag.png)

***Terminate the node and remove it from the cluster***

When you tagged the node unhealthy, the node need to be terminated and removed from the cluster manually with `resize.sh`.

```
/opt/oci-hpc/bin/resize.sh remove --nodes <instance_name>
```

you can check the status and progress of the node removal with below command.

```
tail  -f /opt/oci-hpc/logs/resize_<instance_pool_name>.log
```

***Node Repair Process***

If the node is terminated with “unhealthy” tag on the “CustomerReportedHostStatus” tag name, The data center operation (DO) repair ticket will be generated automatically and then the node will be repaired by datacenter operation team.
Below flow is the repair process according to the status change of the node.

![node_repair_process](images/04_Troubleshooting/05_node_repair_process.png)

## Reference
