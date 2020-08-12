# FAQs of MasQ (SIGCOMM '20)

**Q: How many QPCs can the RDMA NIC store? Is it ever a problem that the VM’s QPC will be switched out, and it will have to request it from the host?**

**A:** The maximum number of QPCs cached in the RNIC and the caching strategy remains unchanged in MasQ and depends on the specific RNIC (I don’t figure out the concrete number). As shown in many works, RDMA itself has the scalability issue, ie, it’s performance drops when the created QPs exceed the maximum number the RNIC can cache. MasQ neither solves this problem nor makes it worse because MasQ doesn’t introduce any storage overhead on RNIC’s SRAM.

**Q: When doing the send operation, how does the RDMA NIC know which QPC to use? because MasQ is not placed on the critical path of sending, I assume it would still use the VM dst IP rather than the Host dst IP which it wont know to match?**

**A:** MasQ uses a QP number (QPN) to match the QPC during data transmission. In RDMA, QPN is an ID to identify the QPC. It is returned to the application after the QPC is successfully created.  Then, during data transmission, it will be carried in each data sending request so that the RNIC can find the QPC to process the application data.

**Q: Can hardware-based solutions solve the problem? What is MasQ different from them?**

**A:** We definitely can offload the network virtualization logic to the hardware. However, hardware-based solutions need special hardware like SmartNIC or FPGA, while MasQ is a sort of software solution. So, MasQ is more cost efficient and could be applied with almost all kinds of RoCE NICs.

Besides, hardware-based solutions potentially have scalability issues. For example, the RNIC has to cache the contexts of virtual networks, such as the VXLAN tunnel table, to realize network virtualization, but the on-chip memory is usually limited. So, if VPC network is large, it will hurt the communication performance since RNIC has to frequently fetch contexts from DRAM.

**Q: How the packets are further routed to the right virtual machine after they arrived at the destination host server? The method replacing the virtual IP address to the host IP address only ensures that the packets can be routed on the host network and delivered to the right destination host server. How about then after that?**

**A:** First, when using RDMA to communicate, two sides in the communication should first create a QP endpoint. Second, the QP number (QPN) is used to identify the endpoint and will be carried in each network packet. So, after packets arrived at the destination host server, we further use the QP number (QPN) to find the corresponding QP endpoint and then write the data into the virtual machine who owns the QP endpoint.

**Q: Whether you have deployed your proposed MasQ on CX4 or newer RNIC? If not, what are the difficulties?**

**A:** No, we haven't deployed MasQ on newer Mellanox RNIC (>=CX4) but we can. To be clear, MasQ's backend can support any OFED-compatible RNICs because it is built on the standard kernel Verbs APIs provided by the `ib_core` module of the OFED. However, a little effort is needed to expose RNIC's full power (mainly the vendor-specific features like Mellanox's BlueFlame for fast IO) to the virtual machine. We must implement a virtual device provider (eg a virtual mlx4 provider for CX3, a virtual mlx5 provider for CX4 and above) in the virtual machine for each type of RNIC (all RNIC types can refer to this driver list, https://github.com/torvalds/linux/tree/master/drivers/infiniband/hw). So, to deploy MasQ on newer Mellanox RNICs, we just need to implement a virtual mlx5 provider in the virtual machine.

MasQ's prototype only implemented a virtual mlx4 provider in the virtual machine because it is the default version when we started this work (three years ago). 

**Q: How does MasQ provide isolation if PFC punishes innocent tenants and, if without PFC, do MasQ's properties still hold?**

**A:** PFC is really a big challenge for RDMA, but we think it should be addressed by congestion control algorithms and is out of scope of this paper. Recently, some new algorithms, such as HPCC, have been proposed. According to their statements, the PFC problem has been largely avoided. Since MasQ is or'thogonal to them, any advanced congestion control algorithm could be used and all MasQ’s good features still hold.

**Q: What's the different with HyV?**

**A:** Compared with HyV, MasQ adopts a similar hybrid I/O virtualization strategy, and this is the main reason that we just briefly discussed the I/O virtualization in the original paper. 

Since HyV is also a software solution, it is definitely possible to extend HyV to enable network virtualization. However, this task is not straightforward and is exactly the main contribution of MasQ. For example, MasQ is the first to provide a virtual RoCE device in guest VMs, where “RoCE” means that application could identify both Ethernet and IB interfaces by a single IP address. This feature is very important, otherwise widely used RDMA_CM APIs will not be available to applications. Furthermore, MasQ is also the first work that tackles the challenge of applying security rules to RDMA networks.

**Q: What's the different with Slim?**

**A:** Slim first proposed connection-based network virtualization, and applied it to the virtual TCP/IP network. MasQ leverages the similar idea for network virtualization, but first applies it to RDMA networking. What's more, RDMA brings more challenges for realizing features required by the VPC because RDMA's data communication fullly bypasses the host software.

**Q: How are network-level techniques implemented in MasQ?** **The VM IPs are translated to host IPs at setup time, and then on the wire only host IPs are used. Guest VMs are not carried inside the RDMA packets.**

**A:** It depends. If the switches in the network have a global view of the network, then it can fetch the virtual network's information from MasQ's controller because MasQ can maintain all information there. If the switches in the network work stand-alone, MasQ is not compatible with them. However, we can enforce the policies in the end host.

**Q: Is it possible that one VM maliciously read/write memory that does not belong to it, leveraging the RNIC? I am also worried about the security of VM isolation on the same hosts because the WQE could directly command the NICs to read/write data based on physical addresses.**

**A:** We rely on RDMA's security mechanisms to protect user memory. First, RDMA resources, such as QP, MR and PD, are created by the backend driver of MasQ, then one VM cannot manipulate resources belonging to other VMs. Second, to communicate with a remote QP, a connection should be established in RC mode or a Q-Key is required in UD mode. Thus, illegal requests could be easily identified and denied in this phase. Third, to correctly access a remote MR, a memory key is required as well as that the remote QP and MR belong to a same PD. Considering above three pre-conditions, we believe that VM’s user memory is properly protected. We will include the discussion about this problem in the revised paper.

**Q: What about smaller data transfers? In that case, how much overhead is introduced?**

**A:** Since most RDMA-based applications, such as HPC and distributed machine learning applications, maintain long-lived RDMA connections, the connection overhead has little effect on the application’s overall performance regardless of the message size. However, for short connections, it indeed takes slightly longer (∼11% in our test cast) time to establish an RDMA connection over MasQ than that over SR-IOV based solutions. 

