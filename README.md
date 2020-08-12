# FAQs of MasQ (SIGCOMM '20)

**Q: How many QPCs can the RDMA NIC store? Is it ever a problem that the VM’s QPC will be switched out, and it will have to request it from the host?**

**A:** The maximum number of QPCs cached in the RNIC and the caching strategy remains unchanged in MasQ and depends on the specific RNIC (I don’t figure out the concrete number). As shown in many works, RDMA itself has the scalability issue, ie, it’s performance drops when the created QPs exceed the maximum number the RNIC can cache. MasQ neither solves this problem nor makes it worse because MasQ doesn’t introduce any storage overhead on RNIC’s SRAM.

**Q: When doing the send operation, how does the RDMA NIC know which QPC to use? because MasQ is not placed on the critical path of sending, I assume it would still use the VM dst IP rather than the Host dst IP which it wont know to match?**

**A:** MasQ uses a QP number (QPN) to match the QPC during data transmission. In RDMA, QPN is a number to identify the QPC. It is returned to the application after the QPC is correctly created.  During data transmission, it will be carried in each data sending request so that the RNIC can find the right QPC to process the application data.

**Q: Can hardware-based solutions solve the problem? What is MasQ different from them?**

**A:** We definitely can offload the network virtualization logic to the hardware. However, hardware-based solutions need special hardware like SmartNIC or FPGA, while MasQ is a software solution. So, MasQ is more cost efficient and could be applied with almost all kinds of RoCE NICs.

Besides, hardware-based solutions potentially have scalability issues. For example, RNIC has to cache the contexts of virtual networks, such as the VXLAN tunnel table, to realize network virtualization, but the on-chip memory is usually limited. So, if VPC network is large, it will hurt the communication performance since RNIC has to frequently fetch contexts from DRAM.

**Q: How the packets are further routed to the right virtual machine after they arrived at the destination host server? The method replacing the virtual IP address to the host IP address only ensures that the packets can be routed on the host network and delivered to the right destination host server. How about then after that?**

**A:** MasQ uses the QP number (QPN) to further route the packets to the destination virtual machine. In RDMA, a QP endpoint on the host is iden

**Q: Whether you have deployed your proposed MasQ on CX4 or newer RNIC? If not, what are the difficulties?**

**A:** No, we haven't deployed MasQ on newer Mellanox RNIC (>=CX4) but we can. To be clear, MasQ's backend can support any OFED-compatible RNICs because it is built on the standard kernel Verbs APIs provided by the `ib_core` module of the OFED. However, a little effort is needed to expose RNIC's full power (mainly the vendor-specific features like Mellanox's BlueFlame for fast IO) to the virtual machine. We must implement a virtual device provider (eg a virtual mlx4 provider for CX3, a virtual mlx5 provider for CX4 and above) in the virtual machine for each type of RNIC (all RNIC types can refer to this driver list, https://github.com/torvalds/linux/tree/master/drivers/infiniband/hw). So, to deploy MasQ on newer Mellanox RNICs, we just need to implement a virtual mlx5 provider in the virtual machine.

MasQ's prototype only implemented a virtual mlx4 provider in the virtual machine because it is the default version when we started this work (three years ago). 

**Q: How does MasQ provide isolation if PFC punishes innocent tenants and, if without PFC, do MasQ's properties still hold?**

**A:** PFC is really a big challenge for RDMA, but we think it should be addressed by congestion control algorithms and is out of scope of this paper. Recently, some new algorithms, such as HPCC, have been proposed. According to their statements, the PFC problem has been largely alleviated. Since MasQ is orthogonal to them, any advanced algorithm could be used and all MasQ’s good features still hold. We will discuss this problem in the revised paper.