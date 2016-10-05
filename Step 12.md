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

   
5. On **node1**, run full chef-client with adapted recipes, so

  * waiting for sync marks is skipped (see proposal https://github.com/crowbar/crowbar-ha/pull/146)
  * when creating new pacemaker resources (in the same transaction for each resource) add a special location constraint that prevents starting those resources on non-upgraded node(s)
  
6. Manually promote DRBD on **node1** to master
