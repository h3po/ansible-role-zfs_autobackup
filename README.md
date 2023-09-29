h3po.zfs_autobackup
=========

:warning: This role is WIP and largely untested

This ansible role sets up zfs autobackup on source and target machines with restricted user accounts, zfs permissions and systemd units.


Role Variables
--------------

Required variables
| variable | default value | description/alternatives
| --- | --- | --- |
| zfs_autobackup_user | zfs-autobackup | the user who will perform the backup
| zfs_autobackup_target_host |  | the host to send the backups to
| zfs_autobackup_source_permissions | snapshot,destroy,hold,release,send,mount,create | permissions of the zfs_autobackup_user on the source
| zfs_autobackup_target_permissions | destroy,hold,release,receive,mount,create | permissions of the zfs_autobackup_user on the target
| zfs_autobackup_timer_schedule | "*-*-* 07:00:00" | systemd timer schedule for the syncronization
| zfs_autobackup_job_name | | Job name
| zfs_autobackup_source_datasets | | dict of datasets to backup or not like { rpool: "true", rpool/backups: "false" }
| zfs_autobackup_target_dataset | | dataset where the backup will be stored

Optional variables
| variable | default value | description/alternatives
| --- | --- | --- |
| zfs_autobackup_pre_snapshot_commands | [] | list of commands to do pre snapshots
| zfs_autobackup_job_parameters | | extra parameters for zfs-autobackup
| zfs_autobackup_source_permissionroots | | list of root datasets for permissions


Example Playbook
----------------

Here's an example to create a backupjob with `host1` as source and `host2` as target. It will send the dataset `rpool` to the target `tank/host2/backups/zfs-autobackup/host1`


```yaml
---
- name: example playbook
  hosts: host1
  become: true
  vars:
    zfs_autobackup_user: zfs-autobackup
    zfs_autobackup_target_host: host2
    zfs_autobackup_job_name: "{{ zfs_autobackup_target_host.split('.')[0] }}"
    zfs_autobackup_source_datasets:
      rpool: "true"
      rpool/backups: "false"
    zfs_autobackup_source_permissionroots:
      - rpool
    zfs_autobackup_target_dataset: "tank/host2/backups/zfs-autobackup/{{ inventory_hostname_short }}"
    zfs_autobackup_pre_snapshot_commands:
      - /usr/local/lib/zfs-autobackup/backup_bootpartition.sh
    zfs_autobackup_job_parameters:
      ssh-target: "{{ zfs_autobackup_user }}@{{ zfs_autobackup_target_host }}"
      keep-source: 1
      keep-target: 1d1w,1w1m,1m1y,1y99y
      buffer: 1G
      destroy-missing: 1s
      debug: ""
  tasks:
    - name: create the zfs backup volume for the efi partition
      community.general.zfs:
        name: rpool/efi-backup
        state: present
        extra_zfs_properties:
          volsize: 512M

    - name: create the boot partition backup script
      ansible.builtin.copy:
        dest: /usr/local/lib/zfs-autobackup/backup_bootpartition.sh
        owner: root
        mode: 0744
        content: |
          #!/bin/sh
          EFIPART="/dev/disk/by-uuid/$(pve-efiboot-tool status | grep "is configured with" | head -n1 | cut -d" " -f1)"
          EFIDISK="$(lsblk -ndo pkname ${EFIPART})"
          /usr/sbin/sgdisk -b /boot/sgdisk_partitiontable_${EFIDISK}.bin /dev/${EFIDISK}
          /usr/bin/dd if=${EFIPART} of=/dev/zvol/rpool/efi-backup bs=1M

    - name: setup zfs-autobackup on source
      ansible.builtin.import_role:
        name: h3po.zfs_autobackup
      vars:
        zfs_autobackup_role: source

    - name: setup zfs-autobackup target
      ansible.builtin.import_role:
        name: h3po.zfs_autobackup
      delegate_to: host2
      vars:
        zfs_autobackup_role: target
```

License
-------

[GNU General Public License v3.0](LICENSE)
