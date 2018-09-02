.. include:: ../global.rst.inc
.. highlight:: none

Backing up entire disk images or disks into disk images
=======================================================

Basically, you can of course either backup disk images like normal files, or backup a disk device directly by piping it to borgbackup. This could be dd, ntfsclone or any other utility that supports stdout/stdin. The simplest use would be `dd if=/dev/somedevice | borg create -pv remote:repository::archive -` where the important part is the dash at the end to read from stdin/the pipe. You should also take care that you either backup something else first or set the environment variable ``

Backing up disk images or disks into disk images can still be efficient with Borg because its `deduplication`_
technique makes sure only the modified parts of the file are stored. Borg also has optional simple sparse file support for extract.

Decreasing the size of image / disk backups
------------------------------------

Disk images are as large as the full disk when uncompressed and might not get much smaller post-deduplication after heavy use because virtually all file systems don't actually delete file data on disk but instead delete the filesystem entries referencing the data. Therefore, if a disk nears capacity and files are deleted again, the change will barely decrease the space it takes up when compressed and deduplicated. Depending on the filesystem, there are several ways to decrease the size of a disk image. Even in general, a more specialzed utility might be preferable to the versatile but blunt `dd`.

Using ntfsclone (NTFS, i.e. Windows VMs)
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

``ntfsclone`` for ntfs filesystems can only operate on filesystems with the journal cleared (i.e. turned-off machines). This also means hibernated Windows installations do not work and if in doubt a disk check should be forced or `ntfsclone` might refuse work, as opposed to `dd` which does not care about the state of the filesystem. This somewhat limits its utility in the case of VM snapshots. However, when it can be used, its special image format is even more efficient than just zeroing and deduplicating.

Simple usage would be `ntfsclone -so - /dev/device | borg create repo::hostname-windows -` where the `-s` tells it to use the special image format and the `-o` followed by a dash in each case denote stdout/stdin. For a more complete backup script that tries to backup all ntfs partitions in addition to disk headers, you could use a script like the following. (one complication in both cases is that ntfsclone/dd and similar tools require root or similar access, which means that either your user needs equivalent access or you need to copy your ssh keys for a remote repository to the root user. Additionally, you might need to set  `BORG_UNKNOWN_UNENCRYPTED_REPO_ACCESS_IS_OK=yes` as borg gives a warning and asks if it should continue if accessing a previously unknown repository, but as you are using stdin the answer prompt would get answered by stdin.)

To automatically save the disk header and the contents of each partition::

    HEADER_SIZE=$(sfdisk -lo Start $DISK | grep -A1 -P 'Start$' | tail -n1 | xargs echo)
    PARTITIONS=$(sfdisk -lo Device,Type $DISK | sed -e '1,/Device\s*Type/d')
    dd if=$DISK count=$HEADER_SIZE | borg create repo::hostname-partinfo -
    echo "$PARTITIONS" | grep NTFS | cut -d' ' -f1 | while read x; do
        PARTNUM=$(echo $x | grep -Eo "[0-9]+$")
        ntfsclone -so - $x | borg create repo::hostname-part$PARTNUM -
    done
    # to backup non-NTFS partitions as well:
    echo "$PARTITIONS" | grep -v NTFS | cut -d' ' -f1 | while read x; do
        PARTNUM=$(echo $x | grep -Eo "[0-9]+$")
        borg create --read-special repo::hostname-part$PARTNUM $x
    done

Restoration is a similar process::

    borg extract --stdout repo::hostname-partinfo | dd of=$DISK && partprobe
    PARTITIONS=$(sfdisk -lo Device,Type $DISK | sed -e '1,/Device\s*Type/d')
    borg list --format {archive}{NL} repo | grep 'part[0-9]*$' | while read x; do
        PARTNUM=$(echo $x | grep -Eo "[0-9]+$")
        PARTITION=$(echo "$PARTITIONS" | grep -E "$DISKp?$PARTNUM" | head -n1)
        if echo "$PARTITION" | cut -d' ' -f2- | grep -q NTFS; then
            borg extract --stdout repo::$x | ntfsclone -rO $(echo "$PARTITION" | cut -d' ' -f1) -
        else
            borg extract --stdout repo::$x | dd of=$(echo "$PARTITION" | cut -d' ' -f1)
        fi
    done

.. note::

   When backing up a disk image (as opposed to a real block device), mount it as
   a loopback image to use the above snippets::

       DISK=$(losetup -Pf --show /path/to/disk/image)
       # do backup as shown above
       losetup -d $DISK

Using zerofree (ext2, ext3, ext4)
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

``zerofree`` works similarly to ntfsclone in that it zeros out unused chunks of the FS,
except it works in place, zeroing the original partition. This makes the backup process
a bit simpler::

    sfdisk -lo Device,Type $DISK | sed -e '1,/Device\s*Type/d' | grep Linux | cut -d' ' -f1 | xargs -n1 zerofree
    borg create --read-special repo::hostname-disk $DISK

Because the partitions were zeroed in place, restoration is only one command::

    borg extract --stdout repo::hostname-disk | dd of=$DISK

.. note:: The "traditional" way to zero out space on a partition, especially one already
          mounted, is to simply ``dd`` from ``/dev/zero`` to a temporary file and delete
          it. This is ill-advised for the reasons mentioned in the ``zerofree`` man page:

          - it is slow
          - it makes the disk image (temporarily) grow to its maximal extent
          - it (temporarily) uses all free space on the disk, so other concurrent write actions may fail.

Virtual machines
----------------

If you use non-snapshotting backup tools like Borg to back up virtual machines, then
the VMs should be turned off for the duration of the backup. Backing up live VMs can
(and will) result in corrupted or inconsistent backup contents: a VM image is just a
regular file to Borg with the same issues as regular files when it comes to concurrent
reading and writing from the same file.

For backing up live VMs use filesystem snapshots on the VM host, which establishes
crash-consistency for the VM images. This means that with most file systems (that
are journaling) the FS will always be fine in the backup (but may need a journal
replay to become accessible).

Usually this does not mean that file *contents* on the VM are consistent, since file
contents are normally not journaled. Notable exceptions are ext4 in data=journal mode,
ZFS and btrfs (unless nodatacow is used).

Applications designed with crash-consistency in mind (most relational databases like
PostgreSQL, SQLite etc. but also for example Borg repositories) should always be able
to recover to a consistent state from a backup created with crash-consistent snapshots
(even on ext4 with data=writeback or XFS). Other applications may require a lot of work
to reach application-consistency; it's a broad and complex issue that cannot be explained
in entirety here.

Hypervisor snapshots capturing most of the VM's state can also be used for backups and
can be a better alternative to pure file system based snapshots of the VM's disk, since
no state is lost. Depending on the application this can be the easiest and most reliable
way to create application-consistent backups.

Borg doesn't intend to address these issues due to their huge complexity and
platform/software dependency. Combining Borg with the mechanisms provided by the platform
(snapshots, hypervisor features) will be the best approach to start tackling them.
