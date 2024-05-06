---
title: CSI Inline Volume Become Orphan After Kubelet Restart When Pod Terminating
update: 2024-05-06 20:25:47
tags:
- kubernetes
categories:
- cloud
---

## CSI Inline Volume Become Orphan After Kubelet Restart When Pod Terminating


### Problem

When the pod is terminating and csi inline volume is deleted, the kubelet down or restart impact the volume become orphan and pod skip delete and unmount the volume. The error message show as follow:


```
Mar 12 00:28:56 tess-node-kltc4-tess19.stratus.lvs.ebay.com kubelet[1526549]: I0312 00:28:56.150307 1526549 state_mem.go:36] "Initialized new in-memory state store"
Mar 12 00:28:56 tess-node-kltc4-tess19.stratus.lvs.ebay.com kubelet[1526549]: E0312 00:28:56.153369 1526549 server.go:302] "Failed to run kubelet" err="failed to run Kubelet: unable to determine runtime API version: rpc error: code = Unknown desc = server is not initialized yet"
Mar 12 00:28:56 tess-node-kltc4-tess19.stratus.lvs.ebay.com systemd[1]: kubelet.service: Main process exited, code=exited, status=1/FAILURE
Mar 12 00:28:56 tess-node-kltc4-tess19.stratus.lvs.ebay.com systemd[1]: kubelet.service: Failed with result 'exit-code'.
Mar 12 00:28:58 tess-node-kltc4-tess19.stratus.lvs.ebay.com systemd[1]: kubelet.service: Scheduled restart job, restart counter is at 1.
Mar 12 00:28:58 tess-node-kltc4-tess19.stratus.lvs.ebay.com systemd[1]: Stopped Kubernetes Kubelet Server.
Mar 12 00:28:58 tess-node-kltc4-tess19.stratus.lvs.ebay.com systemd[1]: Started Kubernetes Kubelet Server.
...
Mar 12 00:29:01 tess-node-kltc4-tess19.stratus.lvs.ebay.com kubelet[1526901]: I0312 00:29:01.976957 1526901 reconciler.go:388] "Could not construct volume information, cleaning up mounts" podName=1517b38e-fa84-4138-b6c0-06663741e385 volumeSpecName="data" err="failed to GetVolumeName from volumePlugin for volumeSpec \"data\" err=kubernetes.io/csi: plugin.GetVolumeName failed to extract volume source from spec: unexpected api.CSIVolumeSource found in volume.Spec"
Mar 12 00:29:01 tess-node-kltc4-tess19.stratus.lvs.ebay.com kubelet[1526901]: I0312 00:29:01.976982 1526901 reconciler.go:421] "Reconciler sync states: could not find volume information in desired state, clean up the mount points" podName=1517b38e-fa84-4138-b6c0-06663741e385 volumeSpecName="data"
Mar 12 00:29:01 tess-node-kltc4-tess19.stratus.lvs.ebay.com kubelet[1526901]: E0312 00:29:01.981982 1526901 operation_generator.go:952] UnmountVolume.MarkVolumeMountAsUncertain failed for volume "" (UniqueName: "data") pod "1517b38e-fa84-4138-b6c0-06663741e385" (UID: "1517b38e-fa84-4138-b6c0-06663741e385") : no volume with the name "data" exists in the list of attached volumes
Mar 12 00:29:01 tess-node-kltc4-tess19.stratus.lvs.ebay.com kubelet[1526901]: E0312 00:29:01.982029 1526901 nestedpendingoperations.go:335] Operation for "{volumeName:data podName:1517b38e-fa84-4138-b6c0-06663741e385 nodeName:}" failed. No retries permitted until 2024-03-12 00:29:02.482017035 -0700 -07 m=+1.794257422 (durationBeforeRetry 500ms). Error: UnmountVolume.TearDown failed for volume "" (UniqueName: "data") pod "1517b38e-fa84-4138-b6c0-06663741e385" (UID: "1517b38e-fa84-4138-b6c0-06663741e385") : kubernetes.io/csi: Unmounter.TearDownAt failed: rpc error: code = Aborted desc = NodeUnpublish operation for volume csi-c6b0a910c881c66f817829d6c0815f60d2dee9c028369214c6f1224be8139978 still ongoing
Mar 12 00:29:44 tess-node-kltc4-tess19.stratus.lvs.ebay.com kubelet[1526901]: I0312 00:29:44.655173 1526901 kubelet.go:2126] "SyncLoop DELETE" source="api" pods=[kube-system/fio-test-6b7df68464-r66bv]
Mar 12 00:29:44 tess-node-kltc4-tess19.stratus.lvs.ebay.com kubelet[1526901]: I0312 00:29:44.658223 1526901 kubelet.go:2120] "SyncLoop REMOVE" source="api" pods=[kube-system/fio-test-6b7df68464-r66bv]
```


From the error messages, we can know what happened:


```

Kubelet restart 
  -> reconstructVolume 
     -> get csi volume plugin by FindAttachablePluginByName and FindDeviceMountablePluginByName 
        -> util.GetUniqueVolumeNameFromSpec 
          -> volumePlugin.GetVolumeName 
             -> getPVSourceFromSpec 
                -> get error "unexpected api.CSIVolumeSource found in volume.Spec" because this function only check CSIPersistentVolumeSource
```


When reconstructVolume gets an error, the volume can not be added into the ActualStateOfWorld in kubelet, then the kubelet will skip this volume unmount when pod deleting.


```
	reconstructedVolume, err := rc.reconstructVolume(volume)
		if err != nil {
			if volumeInDSW {
				// Some pod needs the volume, don't clean it up and hope that
				// reconcile() calls SetUp and reconstructs the volume in ASW.
				klog.V(4).InfoS("Volume exists in desired state, skip cleaning up mounts", "podName", volume.podName, "volumeSpecName", volume.volumeSpecName)
				continue
			}
			// No pod needs the volume.
			klog.InfoS("Could not construct volume information, cleaning up mounts", "podName", volume.podName, "volumeSpecName", volume.volumeSpecName, "err", err)
			rc.cleanupMounts(volume)
			continue
		}

```


So there is a bug that the csi ephemeral volume should go into **‘GetUniqueVolumeNameFromSpecWithPod’** not **‘GetUniqueVolumeNameFromSpec’. **That means csi ephemeral volume **should not** get the attachablePlugin and deviceMountablePlugin.


```
	if attachablePlugin != nil || deviceMountablePlugin != nil {
		uniqueVolumeName, err = util.GetUniqueVolumeNameFromSpec(plugin, volumeSpec)
		if err != nil {
			return nil, err
		}
	} else {
		uniqueVolumeName = util.GetUniqueVolumeNameFromSpecWithPod(volume.podName, plugin, volumeSpec)
	}
```


This place should use `_FindDeviceMountablePluginBySpec_` not `_FindAttachablePluginByName_` to get the volume plugin:


```
	attachablePlugin, err := rc.volumePluginMgr.FindAttachablePluginByName(volume.pluginName)
	if err != nil {
		return nil, err
	}
	deviceMountablePlugin, err := rc.volumePluginMgr.FindDeviceMountablePluginByName(volume.pluginName)
	if err != nil {
		return nil, err
	}

```


Actually, due to pod is terminating and pod not added into the DSW, when reconstructVolume failed, the kubelet still try to unmount and clean mountpoint for this volume:


```
func (dswp *desiredStateOfWorldPopulator) findAndAddNewPods() {
	// Map unique pod name to outer volume name to MountedVolume.
	mountedVolumesForPod := make(map[volumetypes.UniquePodName]map[string]cache.MountedVolume)
	if utilfeature.DefaultFeatureGate.Enabled(features.ExpandInUsePersistentVolumes) {
		for _, mountedVolume := range dswp.actualStateOfWorld.GetMountedVolumes() {
			mountedVolumes, exist := mountedVolumesForPod[mountedVolume.PodName]
			if !exist {
				mountedVolumes = make(map[string]cache.MountedVolume)
				mountedVolumesForPod[mountedVolume.PodName] = mountedVolumes
			}
			mountedVolumes[mountedVolume.OuterVolumeSpecName] = mountedVolume
		}
	}

	processedVolumesForFSResize := sets.NewString()
	for _, pod := range dswp.podManager.GetPods() {
		if dswp.podStateProvider.ShouldPodContainersBeTerminating(pod.UID) {
			// Do not (re)add volumes for pods that can't also be starting containers
			continue
		}
		dswp.processPodVolumes(pod, mountedVolumesForPod, processedVolumesForFSResize)
	}


func (rc *reconciler) cleanupMounts(volume podVolume) {
	klog.V(2).InfoS("Reconciler sync states: could not find volume information in desired state, clean up the mount points", "podName", volume.podName, "volumeSpecName", volume.volumeSpecName)
	mountedVolume := operationexecutor.MountedVolume{
		PodName:             volume.podName,
		VolumeName:          v1.UniqueVolumeName(volume.volumeSpecName),
		InnerVolumeSpecName: volume.volumeSpecName,
		PluginName:          volume.pluginName,
		PodUID:              types.UID(volume.podName),
	}
	// TODO: Currently cleanupMounts only includes UnmountVolume operation. In the next PR, we will add
	// to unmount both volume and device in the same routine.
	err := rc.operationExecutor.UnmountVolume(mountedVolume, rc.actualStateOfWorld, rc.kubeletPodsDir)
	if err != nil {
		klog.ErrorS(err, mountedVolume.GenerateErrorDetailed("volumeHandler.UnmountVolumeHandler for UnmountVolume failed", err).Error())
		return
	}
}

```


But in this time, the container was not finished the container termination, so the volume remove failed, this is the last chance to recycle the volume:


```
[CSI local driver]2024/03/29 03:10:59 logging.go:26: Serving /csi.v1.Node/NodeUnpublishVolume: req=volume_id:"csi-17ef0134f9c76245a954e751a3ebdb57a965d6cf5ea66a6f19bc4b6aa37bad11" target_path:"/var/mnt/kubelet/pods/8c76f1db-b963-4cfc-96b7-6f81fd9781f6/volumes/kubernetes.io~csi/data/mount" 
[CSI local driver]2024/03/29 03:10:59 server.go:1111: Looking up volume with id=vg10000_csi-17ef0134f9c76245a954e751a3ebdb57a965d6cf5ea66a6f19bc4b6aa37bad11
[CSI local driver]2024/03/29 03:10:59 lvm.go:830: Executing: /usr/sbin/lvs --reportformat=json --units=b --nosuffix --options=lv_name,lv_size,vg_name,lv_path vg10000/vg10000_csi-17ef0134f9c76245a954e751a3ebdb57a965d6cf5ea66a6f19bc4b6aa37bad11
                  {"lv_name":"vg10000_csi-17ef0134f9c76245a954e751a3ebdb57a965d6cf5ea66a6f19bc4b6aa37bad11", "lv_size":"21474836480", "vg_name":"vg10000", "lv_path":"/dev/vg10000/vg10000_csi-17ef0134f9c76245a954e751a3ebdb57a965d6cf5ea66a6f19bc4b6aa37bad11"}
[CSI local driver]2024/03/29 03:11:03 server.go:1139: Removing volume with id=vg10000_csi-17ef0134f9c76245a954e751a3ebdb57a965d6cf5ea66a6f19bc4b6aa37bad11
[CSI local driver]2024/03/29 03:11:03 lvm.go:830: Executing: /usr/sbin/lvs --reportformat=json --units=b --nosuffix --options=lv_name,lv_size,vg_name,lv_path vg10000/vg10000_csi-17ef0134f9c76245a954e751a3ebdb57a965d6cf5ea66a6f19bc4b6aa37bad11
                  {"lv_name":"vg10000_csi-17ef0134f9c76245a954e751a3ebdb57a965d6cf5ea66a6f19bc4b6aa37bad11", "lv_size":"21474836480", "vg_name":"vg10000", "lv_path":"/dev/vg10000/vg10000_csi-17ef0134f9c76245a954e751a3ebdb57a965d6cf5ea66a6f19bc4b6aa37bad11"}
2024/03/29 03:11:09 Cleanup lv "vg10000_csi-17ef0134f9c76245a954e751a3ebdb57a965d6cf5ea66a6f19bc4b6aa37bad11": StdoutBuf - "Calling mkfs"
2024/03/29 03:11:09 Cleanup lv "vg10000_csi-17ef0134f9c76245a954e751a3ebdb57a965d6cf5ea66a6f19bc4b6aa37bad11": StderrBuf - "mke2fs 1.45.5 (07-Jan-2020)"
2024/03/29 03:11:09 Cleanup lv "vg10000_csi-17ef0134f9c76245a954e751a3ebdb57a965d6cf5ea66a6f19bc4b6aa37bad11": StderrBuf - "/dev/vg10000/vg10000_csi-17ef0134f9c76245a954e751a3ebdb57a965d6cf5ea66a6f19bc4b6aa37bad11 is apparently in use by the system; will not make a filesystem here!"
2024/03/29 03:11:09 Cleanup lv "vg10000_csi-17ef0134f9c76245a954e751a3ebdb57a965d6cf5ea66a6f19bc4b6aa37bad11": StdoutBuf - "Calling wipefs"
2024/03/29 03:11:09 Cleanup lv "vg10000_csi-17ef0134f9c76245a954e751a3ebdb57a965d6cf5ea66a6f19bc4b6aa37bad11": StderrBuf - "wipefs: error: /dev/vg10000/vg10000_csi-17ef0134f9c76245a954e751a3ebdb57a965d6cf5ea66a6f19bc4b6aa37bad11: probing initialization failed: Device or resource busy"
2024/03/29 03:11:09 Cleanup lv "vg10000_csi-17ef0134f9c76245a954e751a3ebdb57a965d6cf5ea66a6f19bc4b6aa37bad11": StdoutBuf - "Quick reset completed"
[CSI local driver]2024/03/29 03:11:09 lvm.go:830: Executing: /usr/sbin/lvremove -f vg10000/vg10000_csi-17ef0134f9c76245a954e751a3ebdb57a965d6cf5ea66a6f19bc4b6aa37bad11
[CSI local driver]2024/03/29 03:11:18 lvm.go:837: stderr: Logical volume vg10000/vg10000_csi-17ef0134f9c76245a954e751a3ebdb57a965d6cf5ea66a6f19bc4b6aa37bad11 contains a filesystem in use.
[CSI local driver]2024/03/29 03:11:18 logging.go:30: /csi.v1.Node/NodeUnpublishVolume failed: err=rpc error: code = Internal desc = rpc error: code = Internal desc = Failed to remove volume: Failed to lvremove lv vg10000/vg10000_csi-17ef0134f9c76245a954e751a3ebdb57a965d6cf5ea66a6f19bc4b6aa37bad11: Logical volume vg10000/vg10000_csi-17ef0134f9c76245a954e751a3ebdb57a965d6cf5ea66a6f19bc4b6aa37bad11 contains a filesystem in use.
```


So the conditions of producing orphan csi inline volume should be:



1. Pod terminating
2. Kubelet restart or shutdown during pod terminating
3. Kubelet reconstructvolume failed and start unmount before container shutdown 


### Solution

The upstream has reported and fixed this issue at 1.25 version: [https://github.com/kubernetes/kubernetes/pull/108997](https://github.com/kubernetes/kubernetes/pull/108997)

I backport the related pr into our kubernetes repo:  [https://github.corp.ebay.com/tess/kubernetes/pull/2289](https://github.corp.ebay.com/tess/kubernetes/pull/2289)

Before the bug fix rollout, we need manually remove the orphan volume, else it might impact the new pod creating local pvc.

After I audit, I only see the orphan volumes in cluster 40. We can do a change to fix them after kubelet rollout. 