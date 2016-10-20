## Upgrading the controller nodes

Short version of https://etherpad.nue.suse.com/p/cloud-upgrade-6-to-7

1. What's the state before going into upgrade?
  * All non-essential services on all nodes are already stopped
  * Remaining are drbd, keystone, neutron
  
2. Select the first node to upgrade
  * Pick DRBD slave
  * Let's assume we have 2 nodes, first node is **node1**, remaining one is **node2**

3. On **node1**:

  3.1. Migrate  all l3 routers off the l3-agent
  
  3.2. Shutdown pacemaker
  
  3.3. Upgrade the packages to Cloud7 and SP2 (*zypper dup*)
  
  3.4. Reboot
  
4. On **node2**, after **node1** is upgraded:
  
   * Stop and delete pacemaker resources, except of drbd and vip:
   ```
   for type in clone group ms primitive; do
     for resource in $(crm configure show | awk "\$1 == \"$type\" && ! (\$2 ~ /drbd|stonith|vip-/) {print \$2}"); do
       crm --wait resource stop $resource
     done
   done
   for type in location clone group ms primitive; do
     for resource in $(crm configure show | awk "\$1 == \"$type\" && ! (\$2 ~ /drbd|stonith|vip-/) {print \$2}"); do
       crm configure delete $resource
     done
   done
   ```
   * **FIXME** ensure that deleting locations this way is correct
   
   * See https://github.com/crowbar/crowbar-core/pull/716

5. Upgrade related pacemaker location constraint

  5.1 Create the *pre-upgrade* role (technically, it's pacemaker node's attribute) and assign it to all controller nodes that are not upgraded yet (**node2**)

   * We could do this probably already during Cloud6 time
   * Or directly via ``crm node attribute ...``, see https://github.com/crowbar/crowbar-core/pull/702 (merged)
  
  5.2 Create a location constraint that does not allow starting service on the node that is has the *pre-upgrade* role
   * See https://github.com/crowbar/crowbar-openstack/pull/562 (merged)
   
6. Remove "pre-upgrade" attribute from **node1** 

  * So the location constraint does not apply for upgraded node
  * https://github.com/crowbar/crowbar-core/pull/725
  
7. Cluster founder settings

  7.1. Explicitly mark **node1** as the cluster founder
    * Also remove the founder attribute from **node2** if it is there
    * This is needed because pacemaker starts the services on the founder nodes
    * PR: https://github.com/crowbar/crowbar-ha/pull/152 - note that the method is not yet called from anywhere
    
  7.2. Set ``node['drbd']['rsc']['postgresql']['configured']`` to ``false``, otherwise drbd recipe will notice inconsistency and complain.
    * **FIXME** This might not be needed if we call `drbdadm create-md all` explicitely (see next step) ... ?
  
8. DRBD upgrade

 * recreate metadata of drbd in standby node. "drbdadm create-md all", you can use "-- --force" to skipping input  "yes"
 * currently DRBD service is restarted right after each resource is upgraded (see https://github.com/crowbar/crowbar-ha/blob/master/chef/cookbooks/drbd/providers/resource.rb#L66) but that keeps it in inconsistent state when postgresql resource metadata are up-to-date, while it is still old for rabbitmq)

9. On **node1**, run full chef-client with adapted recipes, so

  * waiting for sync marks is skipped (see proposal https://github.com/crowbar/crowbar-ha/pull/146)
  * when creating new pacemaker resources the services are started on upgraded nodes only (see point **5** how to achieve that)
  
10. Manually promote DRBD on **node1** to master
  * This might not be needed. Once we cleanly shutdown **node2**, promotion should happen automatically.
  
11. Wait until **node1** is ready. Proceed with **node2**
