# Butane file to do disk partitioning for `/var/lib/containers` in root disk

```yaml
# storage.bu
variant: fcos
version: 1.3.0
storage:
  disks:
  - device: /dev/disk/by-id/wwn-UPDATE-ME # user must update - use root disk, i.e the same value as .spec.clusters.nodes.rootDeviceHints
    wipe_table: false
    partitions:
    - label: var-lib-containers
      start_mib: 250000-UPDATE-ME # user must update  e.g value `250000` - WARNING: if the value too small, installation may fail to start.
      size_mib: 0-UPDATE-ME # user must update  e.g value `0` - WARNING: if the value too small, deployments may fail post-installation.
  filesystems:
    - path: /var/lib/containers
      device: /dev/disk/by-partlabel/var-lib-containers
      format: xfs
      wipe_filesystem: true
      with_mount_unit: true
      mount_options:
        - defaults
        - prjquota
```

# Convert Butane to ignition and use it with SiteConfig's `ignitionConfigOverride` (recommended)
1. Convert Butane to Ignition. See reference for the link converter. 
    ```shell
    butane storage.bu
    {"ignition":{"version":"3.2.0"},"storage":{"disks":[{"device":"/dev/disk/by-id/wwn-0x6b07b250ebb9d0002a33509f24af1f62","partitions":[{"label":"var-lib-containers","sizeMiB":0,"startMiB":250000}],"wipeTable":false}],"filesystems":[{"device":"/dev/disk/by-partlabel/var-lib-containers","format":"xfs","mountOptions":["defaults","prjquota"],"path":"/var/lib/containers","wipeFilesystem":true}]},"systemd":{"units":[{"contents":"# Generated by Butane\n[Unit]\nRequires=systemd-fsck@dev-disk-by\\x2dpartlabel-var\\x2dlib\\x2dcontainers.service\nAfter=systemd-fsck@dev-disk-by\\x2dpartlabel-var\\x2dlib\\x2dcontainers.service\n\n[Mount]\nWhere=/var/lib/containers\nWhat=/dev/disk/by-partlabel/var-lib-containers\nType=xfs\nOptions=defaults,prjquota\n\n[Install]\nRequiredBy=local-fs.target","enabled":true,"name":"var-lib-containers.mount"}]}}
    ```
2. Copy the output from and place it in Siteconfig's `.spec.clusters.nodes.ignitionConfigOverride` (same level as `.rootDeviceHints`)
    ```yaml
    spec:
      clusters:
        - nodes:
            - ignitionConfigOverride: '{"ignition":{"version":"3.2.0"},"storage":{"disks":[{"device":"/dev/disk/by-id/wwn-0x6b07b250ebb9d0002a33509f24af1f62","partitions":[{"label":"var-lib-containers","sizeMiB":0,"startMiB":250000}],"wipeTable":false}],"filesystems":[{"device":"/dev/disk/by-partlabel/var-lib-containers","format":"xfs","mountOptions":["defaults","prjquota"],"path":"/var/lib/containers","wipeFilesystem":true}]},"systemd":{"units":[{"contents":"# Generated by Butane\n[Unit]\nRequires=systemd-fsck@dev-disk-by\\x2dpartlabel-var\\x2dlib\\x2dcontainers.service\nAfter=systemd-fsck@dev-disk-by\\x2dpartlabel-var\\x2dlib\\x2dcontainers.service\n\n[Mount]\nWhere=/var/lib/containers\nWhat=/dev/disk/by-partlabel/var-lib-containers\nType=xfs\nOptions=defaults,prjquota\n\n[Install]\nRequiredBy=local-fs.target","enabled":true,"name":"var-lib-containers.mount"}]}}'
   ```

# How to verify 
- During or after installation check with hub if BMH is showing the annotation
   ```shell
   oc get bmh -n my-sno-ns my-sno -ojson | jq '.metadata.annotations["bmac.agent-install.openshift.io/ignition-config-overrides"]
   "{\"ignition\":{\"version\":\"3.2.0\"},\"storage\":{\"disks\":[{\"device\":\"/dev/disk/by-id/wwn-0x6b07b250ebb9d0002a33509f24af1f62\",\"partitions\":[{\"label\":\"var-lib-containers\",\"sizeMiB\":0,\"startMiB\":250000}],\"wipeTable\":false}],\"filesystems\":[{\"device\":\"/dev/disk/by-partlabel/var-lib-containers\",\"format\":\"xfs\",\"mountOptions\":[\"defaults\",\"prjquota\"],\"path\":\"/var/lib/containers\",\"wipeFilesystem\":true}]},\"systemd\":{\"units\":[{\"contents\":\"# Generated by Butane\\n[Unit]\\nRequires=systemd-fsck@dev-disk-by\\\\x2dpartlabel-var\\\\x2dlib\\\\x2dcontainers.service\\nAfter=systemd-fsck@dev-disk-by\\\\x2dpartlabel-var\\\\x2dlib\\\\x2dcontainers.service\\n\\n[Mount]\\nWhere=/var/lib/containers\\nWhat=/dev/disk/by-partlabel/var-lib-containers\\nType=xfs\\nOptions=defaults,prjquota\\n\\n[Install]\\nRequiredBy=local-fs.target\",\"enabled\":true,\"name\":\"var-lib-containers.mount\"}]}}"
   ```

- Once the installation is successful check the SNO disk status
   ```shell
   [root@sno ~]# lsblk 
   NAME   MAJ:MIN RM   SIZE RO TYPE MOUNTPOINTS
   sda      8:0    0 446.6G  0 disk 
   ├─sda1   8:1    0     1M  0 part 
   ├─sda2   8:2    0   127M  0 part 
   ├─sda3   8:3    0   384M  0 part /boot
   ├─sda4   8:4    0 243.6G  0 part /var
   │                                /sysroot/ostree/deploy/rhcos/var
   │                                /usr
   │                                /etc
   │                                /
   │                                /sysroot
   └─sda5   8:5    0 202.5G  0 part /var/lib/containers
   
   [root@sno ~]# df -h
   Filesystem      Size  Used Avail Use% Mounted on
   devtmpfs        4.0M     0  4.0M   0% /dev
   tmpfs           126G   84K  126G   1% /dev/shm
   tmpfs            51G   93M   51G   1% /run
   /dev/sda4       244G  5.2G  239G   3% /sysroot
   tmpfs           126G  4.0K  126G   1% /tmp
   /dev/sda5       203G  119G   85G  59% /var/lib/containers
   /dev/sda3       350M  110M  218M  34% /boot
   tmpfs            26G     0   26G   0% /run/user/1000
   ```

# Convert Butane to MachineConfig and use it with SiteConfig's `extraManifestPath` (optional)
Butane binary can also generate a MachineConfig CR and passed in via `.spec.clusters.extraManifestPath`
```yaml
# storage-mc.bu
variant: openshift
version: 4.13.0
metadata:
  name: 98-var-lib-containers-partitioned
  labels:
    machineconfiguration.openshift.io/role: master
spec: 
  ... # same as original fcos variant
```
Generated file 
```yaml
# Generated by Butane; do not edit
apiVersion: machineconfiguration.openshift.io/v1
kind: MachineConfig
metadata:
  labels:
    machineconfiguration.openshift.io/role: master
  name: 98-var-lib-containers-partitioned
spec:
  config:
    ignition:
      version: 3.2.0
    storage:
      disks:
        - device: /dev/disk/by-id/wwn-0x6b07b250ebb9d0002a33509f24af1f62
          partitions:
            - label: var-lib-containers
              sizeMiB: 0
              startMiB: 250000
          wipeTable: false
      filesystems:
        - device: /dev/disk/by-partlabel/var-lib-containers
          format: xfs
          mountOptions:
            - defaults
            - prjquota
          path: /var/lib/containers
          wipeFilesystem: true
    systemd:
      units:
        - contents: |-
            # Generated by Butane
            [Unit]
            Requires=systemd-fsck@dev-disk-by\x2dpartlabel-var\x2dlib\x2dcontainers.service
            After=systemd-fsck@dev-disk-by\x2dpartlabel-var\x2dlib\x2dcontainers.service

            [Mount]
            Where=/var/lib/containers
            What=/dev/disk/by-partlabel/var-lib-containers
            Type=xfs
            Options=defaults,prjquota

            [Install]
            RequiredBy=local-fs.target
          enabled: true
          name: var-lib-containers.mount
```


# Reference: 
- Butane binary to help with conversion can be found [here](https://coreos.github.io/butane/getting-started/#getting-started).
- Fedora docs with [examples](https://docs.fedoraproject.org/en-US/fedora-coreos/storage/#_setting_up_separate_var_mounts)