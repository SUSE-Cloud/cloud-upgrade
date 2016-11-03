## Upgrading the controller nodes

Short version of https://etherpad.nue.suse.com/p/cloud-upgrade-6-to-7

1. What's the state before going into upgrade?
  * All non-essential services on all nodes are already stopped
  * Remaining are drbd, keystone, neutron
  
2. Select the first node to upgrade
  * Pick DRBD slave
  * Let's assume we have 2 nodes, first node to upgrade (= DRBD slave) is **node1**, remaining one is **node2**

3. On **node1**:

  3.1. Migrate  all l3 routers off the l3-agent
  
  3.2. Shutdown pacemaker
  
  3.3. Upgrade the packages to Cloud7 and SP2 (*zypper dup*)
  
  3.4. Reboot
  
4. On **node2**, after **node1** is upgraded:
  
   * Stop and delete pacemaker resources, except of drbd and vip:

   * **FIXME** keystone does not want to die
   
   * neutron-agents: can be deleted as well (they will be started there though)
   
   * See https://github.com/crowbar/crowbar-core/pull/716

5. Upgrade related pacemaker location constraint

  5.1 Create the *pre-upgrade* role (technically, it's pacemaker node's attribute) and assign it to all controller nodes that are not upgraded yet (**node2**)

   * We could do this probably already during Cloud6 time
   * Or directly via ``crm node attribute ...``, see https://github.com/crowbar/crowbar-core/pull/702 (merged)
   * Fixed in https://github.com/crowbar/crowbar-core/pull/749
  
  5.2 Create a location constraint that does not allow starting service on the node that is has the *pre-upgrade* role
   * See https://github.com/crowbar/crowbar-openstack/pull/562 (merged)
     
  5.3. Do not put this location constraint to `neutron-agents`, `ms-drbd-postgresql-controller` and `ms-drbd-rabbitmq-controller` (see `postgresql/recipes/ha_storage.rb` and `rabbitmq/recipes/ha.rb`)
  
   * See https://github.com/crowbar/crowbar-openstack/pull/567
   
  5.4. **TODO** Figure out how to handle neutron-agents correctly
   
    5.4.1. We need to allow starting `neutron-agents` on **node2** so they have access to routers that are present there.
   
    5.4.2. We need to migrate routers again from non-upgraded node to **node1** once all services are running with new configuration at **node1**
   
    This could be achieved by not adding any constraint to `neutron-agents` resource. First chef-client on **node1** would start `nutron-agents` on both nodes. But:
   
    5.4.3. Once we upgrade **node2**, we can't allow starting neutron-agents there, before the configuration is updated, i.e. before the chef-client run on **node2** is finished. So it looks like for this time, we need a constraint that allows `neutron-agents` to be running at upgraded nodes only.
   
  
6. Remove "pre-upgrade" attribute from **node1** 

  * So the location constraint does not apply for upgraded node
  
    * See https://github.com/crowbar/crowbar-core/pull/725
    
  * When could we do it and from where? Probably from the **node2**, which still has pacemaker running
  
  
7. Cluster founder settings

  7.1. Explicitly mark **node1** as the cluster founder
    * Also remove the founder attribute from **node2** if it is there
    * This is needed because pacemaker starts the services on the founder nodes
    * PR: https://github.com/crowbar/crowbar-ha/pull/152 - note that the method is not yet called from anywhere
    
  7.2. Set ``node['drbd']['rsc']['postgresql']['master']`` to ``true`` for **node1** AND ``false`` for **node2**, otherwise drbd recipe will notice inconsistency and complain.
   
    * See https://github.com/crowbar/crowbar-ha/pull/155
  
8. DRBD upgrade

  * recreate metadata of drbd at **node1** using "drbdadm create-md all", you can use "-- --force" to skipping input  "yes"
  * This has to be done explicitely from script. We cannot do it from chef resource, because DRBD service is restarted right after each resource is upgraded (see https://github.com/crowbar/crowbar-ha/blob/master/chef/cookbooks/drbd/providers/resource.rb#L66) but that keeps it in inconsistent state when postgresql resource metadata are up-to-date, while it is still old for rabbitmq)
 
9. On **node1**, start pacemaker

  * so pacemaker starts DRBD synchronizes it with **node2**.
  * **FIXME** currently this only works after **second** run of `create-md` see  https://bugzilla.suse.com/show_bug.cgi?id=1006105
  * Wait until DRBD is correctly synchronized

10. On **node1**, start crowbar-join that runs chef-client and moves the node to **ready** state

  * waiting for sync marks is skipped (see proposal https://github.com/crowbar/crowbar-ha/pull/146)
  * when creating new pacemaker resources the services are started on upgraded nodes only (see point **5** how to achieve that)
  * only neutron-agents are started on both nodes
  
11. Manually promote DRBD on **node1** to master
  * This should not be needed. Once we cleanly shutdown **node2**, promotion should happen automatically.
  
12. Wait until **node1** is ready. Proceed with **node2**

13. Execute pre-upgrade script at **node2** so
  * neutron routers are migrated off this node
   
    **FIXME** 
    How should we do it? See https://etherpad.nue.suse.com/p/cloud-upgrade-6-to-7 lines 284-287
    
  * pacemaker is stopped
  
14. Upgrade and reboot **node2**
  
