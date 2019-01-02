---
title: repair-thinpool
update: 2018-10-12 14:34:41
tags: lvm
categories: storage
---

# Repair thin-pool document

### Use lvm repair

If you would have latest lvm2 tools - you could have tried:

1. deactive if pool is active, before deactive pool, you must to deactive the volumes created from pool. if volume is created by lvm, just umount volume and lvchange, but if volume is created by devicemapper, need manual remove volume and record the deviceId deviceName and table to activate volume. Deactive, actually, is the process of remove the file link of /dev/vg/volume and* /dev/mapper/vg-volume.

   ```
   lvchange -an vg
   ```

2. repair meta and active pool

   ```
   lvconvert --repair  vg/pool
   ```

3. active pool

   ```shell
   lvchange -ay vg
   ```

   Below are the steps which happen while running the consistency check:

   1. Creates a new, repaired copy of the metadata.
      lvconvert runs the thin_repair command to read damaged metadata from
      the existing pool metadata LV, and writes a new repaired copy to the
      VG’s pmspare LV.
   2. Replaces the thin pool metadata LV.
      If step 1 is successful, the thin pool metadata LV is replaced with
      the pmspare LV containing the corrected metadata. The previous thin
      pool metadata LV, containing the damaged metadata, becomes visible
      with the new name ThinPoolLV_tmetaN (where N is 0,1,…).

   but in my lvm (version 2.02.166(2)-RHEL7 (2016-11-16)), repair will not create new pmspare, it will direct use free vg to create new meta:

   ```shell
   [root@tosqatest4 ~]# lvs -a silver_vg
     LV                                VG        Attr       LSize   Pool                      Origin Data%  Meta%  Move Log Cpy%Sync Convert
     convoy_Linear_silver_data         silver_vg twi-aotz-- 894.25g                                  0.04   0.01                            
     convoy_Linear_silver_data_meta0   silver_vg -wi-a-----   9.31g                                                                         
     [convoy_Linear_silver_data_tdata] silver_vg Twi-ao---- 894.25g                                                                         
     [convoy_Linear_silver_data_tmeta] silver_vg ewi-ao----   9.31g                                                                         
     thinvolume                        silver_vg Vwi-aotz--   1.00g convoy_Linear_silver_data        40.04
   ```

   So pmspare device just a free space device nor a mirror of metadata, if not, we can add pv to vg for repair.

   ### Use manual repair

   With older tools - you need to go in these manual step:

   1. create temporary small LV

      ```
      lvcreate -an -Zn -L10 --name temp vg
      ```

   2. replace pool’s metadata volume with this tempLV

      ```
      lvconvert --thinpool vg/pool  --poolmetadata temp
      ```

   (say ‘y’ to swap)

   1. activate & repair metadata from ‘temp’ volume - you will likely need another volume where to store repaire metadata -

      so create:

      ```shell
      lvcreate -Lat_least_as_big_as_temp  --name repaired  vg
      lvchage -ay vg/temp
      thin_repair -i /dev/vg/temp  /dev/vg/repaired
      ```

      if everything when fine - compare visualy ‘transaction_id’ of repaired metadata (thin_dump /dev/vg/repaired)

      1. swap deactivated repaired volume back to your thin-pool

         ```
         lvchange -an vg/repaired
         lvconvert --thinpool vg/pool --poolmetadata repaired
         ```

      try to activate pool - if it doesn’t work report more problems.

      ## Metadata space exhaustion

      Metadata space exhaustion can lead to inconsistent thin pool metadata
      and inconsistent file systems, so the response requires offline
      checking and repair.

      1. Deactivate the thin pool LV, or reboot the system if this is not

         ```
         possible.
         ```

      2. Repair thin pool with lvconvert –repair.

         ```
         See "Metadata check and repair".
         ```

      3. Extend pool metadata space with lvextend VG/ThinPoolLV_tmeta.

         ```
         See "Manually manage free metadata space of a thin pool LV".
         ```

      4. Check and repair file system with fsck.
