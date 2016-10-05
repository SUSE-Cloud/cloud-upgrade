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
  
   4.1. Stop and delete pacemaker resources, except of drbd and vip:
   ```
   for type in clone group ms primitive; do
     for resource in $(crm configure show | awk "\$1 == \"$type\" && ! (\$2 ~ /drbd|stonith|vip-/) {print \$2}"); do
       crm --wait resource stop $resource
     done
   done
   for type in clone group ms primitive; do
     for resource in $(crm configure show | awk "\$1 == \"$type\" && ! (\$2 ~ /drbd|stonith|vip-/) {print \$2}"); do
       crm configure delete $resource
     done
   done
   ```

5. Upgrade related pacemaker magic

  5.1 Create the *pre-upgrade* role (technically, it's pacemaker node's attribute) and assign it to all controller nodes that are not upgraded yet (**node2**)
   * Use similar code as in https://github.com/crowbar/crowbar-openstack/blob/master/crowbar_framework/lib/openstack/ha.rb#L19
  
  5.2 Create a location constraint that does not allow starting service on the node that is has the *pre-upgrade* role
   * Use similar code as in https://github.com/crowbar/crowbar-openstack/blob/master/chef/cookbooks/crowbar-openstack/libraries/ha_helpers.rb , but place it to crowbar-ha instead
   * Use pre-upgrade eq false condition for indicating that pre-upgrade nodes should not get the service
  
  5.3. Create a definition *controller_before_upgrade_for*
   * similar to *openstack_pacemaker_controller_only_location_for* from https://github.com/crowbar/crowbar-openstack/blob/master/chef/cookbooks/crowbar-openstack/definitions/openstack_pacemaker_controller_only_location_for.rb
   * This is optional, so the following step adds less code to normal (not upgrade related) recipes
  
  5.4. Update ha related OpenStack recipes to add a new location to the same transaction with creation "normal" services.
   * Something like:
  
   ```
    upgrade_location_name = controller_before_upgrade_for clone_name
    transaction_objects << "pacemaker_location[#{upgrade_location_name}]" if CrowbarPacemakerHelper.being_upgraded?(node)
   ```
   
6. On **node1**, run full chef-client with adapted recipes, so

  * waiting for sync marks is skipped (see proposal https://github.com/crowbar/crowbar-ha/pull/146)
  * when creating new pacemaker resources (in the same transaction for each resource) add a special location constraint that prevents starting those resources on non-upgraded node(s)
  
7. Manually promote DRBD on **node1** to master
