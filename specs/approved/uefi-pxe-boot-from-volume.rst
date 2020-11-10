..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

====================================
UEFI PXE boot from root iSCSI volume
====================================


https://storyboard.openstack.org/#!/story/2008023

Currently BFV (Boot From Volume) only supports iPXE not PXE. As not all the
servers support iPXE boot due to lacking of integrated NIC driver support in
iPXE, especially many aarch64 servers don't support iPXE. But all the server
may support PXE boot. So it makes sense that Ironic can support PXE BFV.

Problem description
===================

A detailed description of the problem:

In general, the whole provision procedure of PXE BFV is similar to iPXE BFV's.
The difference Ironic can see is, in the deploying provision state, it involves
use of a ``kernel`` and ``ramdisk`` image which cached from partition user
image and hosted by the conductor host in order to enable the node to boot
initrd via PXE with sufficient command line parameters to enable the root
volume to attach during the initrd boot sequence.

The whole use case and provision procedure of PXE BFV is shown as below, with
differences compared to iPXE BFV:

  - Operator uploads partition user image to Glance image service.

    Differences:
    PXE BFV uses partition user image, and iPXE uses whole disk image.

  - Operator enrolls nodes with property capabilities='boot_mode:uefi,
    iscsi_boot:True, boot_option:netboot' and make them as available provision
    state.

  - End user launches a bare metal instance with a bootable volume and chooses
    the partition user image.

    Differences:
    PXE boots from an existing volume like iPXE is not support, because the
    existing volume doesn't contain any image meta data. This also means that
    PXE BFV only support cinder storage interface and no external storage
    interface support as there is no other way to cache kernel/ramdisk
    currently.

  - Ironic conductor changes the node provision state to deploying from
    available, prepare instance to boot:

      - Cache kernel/ramdisk images.
      - Generate PXE grub config menuentry 'boot_iscsi', fill iSCSI kernel
        command line options which defined by
        `Dracut iSCSI kernel command line option`_ and
        `Initramfs iSCSI kernel command line option`_.
      - Set default boot menuentry to 'boot_iscsi' and default boot interface
        to PXE.

    Differences:
    With iPXE BFV, no need to cache kernel and ramdisk images, volume info
    pass to OS in another way call iBFT (`iSCSI Boot Firmware Table`_) not via
    kernel command line option.

  - Node provision state changed to active from deploying, power on and boot
    defaults to PXE network booting. PXE boot into initrd. Legacy BIOS PXE
    boot is out of scope of this specification.

    Differences:
    PXE loads grub from tftpboot network dir and just boots into initrd, it
    doesn't do any other thing. Whereas iPXE will attach the iSCSI volume disk
    firstly, then load the grub from the volume disk, it also fills the iBFT
    APCPI table with volume info. Finally it boots into initrd.

  - Initrd setup network ip with dhcp request, get the volume info from
    command line options, attach the root volume disk, mount it and finally
    chroot to it, continue booting the OS until it finishes booting.

    Differences:
    PXE BFV gets the volume info from kernel command line options and iPXE BFV
    gets it from the iBFT ACPI table. See `Debian volume attach script`_ as an
    example.

Proposed change
===============

According to above PXE BFV whole provision procedure, Ironic only get involved
before node provision state becomes active. Most of the code flow in Ironic is
the same as iPXE BFV. So upon on current iPXE BFV, to support PXE BFV, it
needs to handle PXE BFV also for the same code flow and add new code logic for
PXE BFV specific. Most likely, the main changes should come into the PXE boot
interface.

The proposed changes are:

  - Update ``pxe.PXEBoot`` capabilities to add 'iscsi_volume_boot.

  - Update ``pxe_utils.build_pxe_config_options`` util logic to support
    generating volume pxe options for PXE BFV also.

  - Update ``pxe_base.PXEBaseMixin.prepare_instance`` driver logic to support
    caching kernel and ramdisk for PXE BFV.

  - Update ``pxe_grub_config.template`` to add menuentry 'boot_iscsi'.

  - Update ``storage.cinder.CinderStorage.validate`` driver logic to support
    PXE.

  - Update ``storage.cinder.CinderStorage.should_write_image`` driver logic to
    handle PXE BFV case which has an image source.

Alternatives
------------

None

Data model impact
-----------------

None

State Machine Impact
--------------------

None

REST API impact
---------------

None

Client (CLI) impact
-------------------

"openstack baremetal" CLI
~~~~~~~~~~~~~~~~~~~~~~~~~

None

"openstacksdk"
~~~~~~~~~~~~~~

None

RPC API impact
--------------

None

Driver API impact
-----------------

None

Nova driver impact
------------------

Currently, Nova Ironic driver doesn't pass image source info to Ironic in
case of BFV. Change needed to make to pass image source info to Ironic in
case of PXE BFV.

Ramdisk impact
--------------

None

Security impact
---------------

Tenant network isolation is unsupported due to the need for PXE boot.

``Note`` that iSCSI volume info like username/password specified on the kernel
command line are visible for all users via the file /proc/cmdline or via
dmesg. Who gets the volume info can login and mount the iSCSI volume in the
same storage network.

Other end user impact
---------------------

PXE BFV doesn’t support whole disk image as iPXE BFV. It only supports
partition user image. Which means that kernel updating automatically is not
supported, It needs to copy kernel/initrd to tftpboot dir for PXE boot
manually when updating kernel.

The user needs to know how to build partition user images. It should be built
with an iscsi-boot element.
e.g.:
disk-image-create -o $IMAGE_NAME $DISTRO baremetal dhcp-all-interfaces \
iscsi-boot

Scalability impact
------------------

None

Performance Impact
------------------

None

Other deployer impact
---------------------

None

Developer impact
----------------

None

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  Xinliang Liu (irc: xinliang, xinliang.liu@linaro.org)

Work Items
----------

* Implement support to pass image source info into Ironic for PXE BFV (Nova).
* Implement support for PXE BFV.
* Update `Boot From Volume user guide`_ page on how to use this functionality.

Dependencies
============

Implementation of
`Nova pass image source info into Ironic for PXE iSCSI boot`_.
Ironic requires it to cache kernel/ramdisk for the PXE BFV user case.


Testing
=======

Unit tests should be sufficient for ensuring this functionality is not broken.

A tempest test for the whole provision procedure is not feasible now due to no
partition user image available on the web site for testing.

Upgrades and Backwards Compatibility
====================================

N/A

Documentation Impact
====================

`Boot From Volume user guide`_ page needs to be updated on how to use this
functionality.

References
==========

Relevant specifications:
  - `Boot from Volume - Reference Drivers`_

.. _`Nova pass image source info into Ironic for PXE iSCSI boot`: https://review.opendev.org/#/c/746626/
.. _`Boot From Volume user guide`: https://docs.openstack.org/ironic/latest/admin/boot-from-volume.html
.. _`Dracut iSCSI kernel command line option`: https://man7.org/linux/man-pages/man7/dracut.cmdline.7.html
.. _`Initramfs iSCSI kernel command line option`: https://salsa.debian.org/linux-blocks-team/open-iscsi/-/blob/master/debian/extra/initramfs.local-top#L248
.. _`iSCSI Boot Firmware Table`: http://download.microsoft.com/download/7/E/7/7E7662CF-CBEA-470B-A97E-CE7CE0D98DC2/iBFT.docx
.. _`Debian volume attach script`: https://salsa.debian.org/linux-blocks-team/open-iscsi/-/blob/master/debian/extra/initramfs.local-top
.. _`Boot from Volume - Reference Drivers`: https://specs.openstack.org/openstack/ironic-specs/specs/9.0/boot-from-volume-reference-drivers.html
