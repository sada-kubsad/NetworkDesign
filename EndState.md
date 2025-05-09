# Table of Contents
<details>
<summary>Options for End State</summary>

- [Single VNet with VPN Termnination on Palo VPN GW (selected)](#single-vnet-with-vpn-termination-on-palo-vpn-gw)
  - [Solutions Details](#)
  - [Architecture diagram](#)
  - [Pros](#)
  - [Cons](#)

- [Single VNet with VPN Termination on Azure VPN GW](#single-vnet-with-vpn-termination-on-azure-vpn-gw)
  - [Architecture Diagram](#)
  - [Pros](#)
  - [Cons](#)

- [Transit/Spoke VNet for VPN](#transitspoke-vnet-for-vpn)
  - [Solution Description](#)
  - [Diagram](#)
  - [Pros](#)
  - [Cons](#)

- [vWAN based Solution](#vwan-based-solution)
  - [Solution Description](#)
  - [Architecture Diagram](#)
  - [Pros](#)
  - [Cons](#)

</details>

<details>
<summary>Selection Criteria for End-State</summary>
</details>

# Options for End State:

## Single VNet with VPN Termination on Palo VPN GW:
### Architecture diagram:
![image](https://github.com/user-attachments/assets/64d57f09-50a5-46c7-91ec-370670141a9f)

### Solutions Details:
- Leverages the Palos to terminating the client/customer VPN tunnels
- If we want to terminate the VPN on the Palos, we can use their static public IPs and bypass the public Load Balancer for IPSec packets.
  - The customer side would have an active-standby configuration (like on Cisco ASA) where you can put multiple peer IPs.
- First deployment of Eagle in Azure leveraged Active-Active Palos peered to ARS. ARS handled all the VPN networks. The customer could determine which Palo they peered to, based off the learnt routes the Palo would advertise to ARS. The VMs would know which Palo has the active tunnel because it would re-advertise that to ARS.
  - The downside was that it was that tunnel building was co-managed. When customers has to build out tunnels to both Palo 1 and Palo 2, but customers failed to build out tunnels on Palo 2. Hence it did not work out in practice.
    - This problem can be address if the client configuration to Palo 1 and Palo 2 were built out using automation.
  - With Cisco ASA, when you can set peer IP1 IP2, makes the client will try IP1, and if it can't it will try IP2.
  - You have to set the secondary tunnel as passive mode because if both tunnels are down and the VM (onprem or AVS) initiates a reach out connection to the customer (like for printing with Bistrack), the VM connection enters passive mode so the customer has to bring up the tunnel by initiating traffic.
    - To keep the tunnel up, customers will have to trigger on their side by setting up IP SLA.
    - Customers may not have the technical ability to bring up the secondary tunnel or do IP SLA.
    - IN AWS, you can download the VPN config which sets up the config such that you can trigger the IP SLA from the client side.
    - P21 has 45 customers that need VPN, not using the web version. But even if using web, they could still use VPN for printing.
    - Customers could build a Cron job on a Linux machine with a loop that pings the other side.
  - Why not clients have tunnels up to both primary and secondary Palos and setup Local Preference?
    - Tunnels are not BGP tunnels, tunnels are static tunnels that are policy based.
  - We have to assume that the client at a minimum knows how to setup a tunnel, but not know how to manage it beyond dynamic routing etc.
- How do we get the default route to AVS? Palo -(BGP advertise)-> ARS -> ER GW -> AVS
  - Palo BGP engine will selective advertise to ARS
- NATing on Palo
  - Client -> Palo VPN -(SNAT) -> ER GW -> AVS
    - VPN coming over 10.0.0.1 from Client 1 and VPN2 is also coming over 10.0.0.1 from Client 2. We need to NAT those IPs on the Palo Alto into a reserved space that we know is always NATed space.
    - The prefix for all the NATed IPs can be advertised to ARS which will then advertise to AVS.
- Move each VPN:
  - then just advertise the prefix of those routes from Palo -> ER GW -> AVS
- Move a server from on-prem to AVS:
  - then just advertise the new route to the migrated server to everyone's VPN
- Internet Traffic:
  - Controlled via default route
  - Can be moved anytime that the VM are ready to change their outbound internet address
  - For a VM that is in AVS that using the default route to go back on-prem to get to the Internet
  - If we change the default route on the NSXT router will that affect every subnet. So we cannot migrate a subnet from on-prem to AVS at a time.
    - All traffic for everything in the AVS environment would have to cut over at the same time for outbound internet access
    - We can have separate NSXTs per subnet, but that would require separate AVS environments
      - In a single AVS/NSXT environment, you can have a single T0 but multiple T1s because you will be BGP peering with T0. 
      - Hence we could go with multiple AVS instances, one for P21, one for BisTrac etc. 
      - There is no limit of how many AVS clusters you can have per region.
    - We could do a big bang cut-over with everyone internet going through Azure
      - Different business units with different solutions and platforms will not co-operate to make that happen.
  - We can utilize the same circuits from Austin and use global reach to connect each AVS environment. 
  - Some apps will remain on-prem while others are in AVS. Hence some apps need Internet egress through Azure while others will Internet egress through on-prem.
- Cutover: 
  - Layer 3 interface for the gateway can be moved with a button in HCX that flips the gateway to AVS.
  - Problem: That is a big bang change that moves all the routing over to AVS. Can't cut over everything at once. Will not have Internet egress when everything cuts over big bang style. Even if we migrated Layer 3 subnet by subnet, the internet egress will cut over big bang over the layer 2 stretch. When these VMs cut over to AVS, they need internet access and they need VPN access.
    - How can we migrate one VPN at a time if we move all routing all at once. If you cut over the Internet for everyone in one shot, you are also cutting over the VPN for everyone in one shot. 
    - Customer has site to site VPN with Austin today, we need to move that traffic to Palo Alto first. But for the VM inside of AVS to be be able to route to the Palo Alto or any IP in that encryption domain, every VM would need static routes pointing to the NSXT as its router. The VMs are not going to use the NXST until the big bang cutover unless using BGP to every single VM.
      - The only thing routing can do is to allow the NSXT to learn the routes. But the VM in AVS is not goign to usee the NSXT for routing.
  - Mobility Optimized Networking (MON) feature in HCX can spoof the default gateway from on-prem and put it in NSXT. So VMs that get migrated into AVS, even though on extended VLAN, we can essentially say the GW is local. So now the traffic does not have to hairpin all the way back to on-Prem. 
    - As of setting up Layer 2 stretch (previous step), you then have a default GW in both on-prem and on T1 router in Azure.
    -  Without MON:
       -  The default GW is on-oprem. When we extend the VLAN, we are taking the default GW to AVS, but not connecting it to the Tier 1 router. Traffic still hairpins back on-prem. 
       -  When you stretch the VLAN, on-prem GW is going to be sitting on the Tier 1 router in an admin down state. On-prem is advertising the entire subnet to Tier 1 router in AVS which is in admin down state.  
    -  With MON:
       -  When you enable MON it brings up (sets admin state up) the GW on the Tier 1 router in AVS. NSXT in going to advertise a /32 per VM with next hop to its specified destination via ER. So now if a VM needs to talk to something, it doesn't have to hairpin back on-prem, it just goes straight out of NSX - it goes from Tier 1 to Tier 0 across ER to its intended destination.
       - MON can influence routing per VM or per subnet. Per VM is the key. 
       - In the old school way, you can BGP peer with the VM itself.
       - MON is to be used for influencing Layer 3 traffic
       - Using MON will require some policy routes created to define where to send the traffic
         - When MON is off, the default policy routes (for RFC 1918 addresses) sends the traffic back to the default GW on-prem
         - When MON is on, a policy per VM IP specifies where to send the traffic: to on-prem GW or to T1 GW in AVS  
       - With MON, if you move 10 VMs on a subnet over to AVS, they are still on the same Layer 2 subnet/VLAN, they are still using the on-prem GW.
         - Schedule a maintenance for Customer A to move their VPN to PAN. At the same time enable MON just for those 10 VMs. Those 10 VMs are now routing with the NSXT, so they would route all traffic (including VPN  and Internet traffic) would go through the VNet if we are letting 0/0 over ER.
       - MON eliminates the need for multiple AV instance because you no longer need to do a big bang routing on the NSXTs per AVs instance. 
- Cutover Summary:
  - Layer 2 cutover is a zero change from a routing perspective.
    - The VM moves over to AVS while routing stays the same. 
  - Cutover of Layer 3 happens with MON on a per VM basis.
    - Also cuts over the VPN and Internet routing from on-prem GW to VNet in Azure
    - Enable MON for the VM being cut over.


### Pros:
- Removes the complexity introduced by the clients/customers receiving all on-prem and AVS routes

### Cons:
- Redundancy and resiliency goes down because of no failover as the Palo's cannot do VPN when frontended by a Load Blancer for redundancy and failover. Microsoft LB does not support IPSec packets required for the VPN tunnels to Palos.


## Single VNet with VPN Termination on Azure VPN GW:

### Architecture Diagram:
![image](https://github.com/user-attachments/assets/5827739e-c81b-44f8-94e8-8e02c0d083da)

### Solution Design:
- Other spokes of the Hub and Spoke solution are dotted lines because we don't know there is a use case yet for needing the spokes.
- Every single VPN is considered East West traffic and will be filtered through the Palo inside interface. The next hop will be the ILB. Then it will forward to one of Palo Altos, and then traffic leaves the same interface to another destination that is explicitly allowed.
  - The Azure VPN gateway will forward to a LB
- VPN gateway in the hub is connected to AVS via the ER circuit and ER GW in the hub.
- With the ER circuit connecting the hub ER GW and AVS, the Hub learns the AVS prefixes
- In a hub and spoke design, ER GW learnt routes will leak to the end customers via the ARS acting as a route reflector between ER Gateway and VPN Gateway.
- NATing:
  - PAN FW will need to SNAT the end customer's traffic to AVS
  - Tunnels will terminate on the VPN GW in the Hub VNet with destination as PAN. When end customers want to communicate with the app hosted on AVS, the PAN will DNAT the traffic and SNAT it.
- Once AVS is connected to the ER GW in the Hub, T0 in AVS will be responsible for advertising whatever it knows in AVS via the ER GW (in the hub VNet) to
  - VNets in Azure
  - On-prem locations
  - Learning:
    - AVS is learning VNets connected to the Hub
    - VNets are learning AVS routes
- We want everything to go through the Palo for outbound (internet or elsewhere)
- ARS is needed to reflect/exchange the routes between the ER GW and VPN GW because by default VPN GW is not learning the routes from ER GW (and everything connected to the ER GW like AVS).
  - It will all be static routing from Palo to the next hop of something in AVS
  - After traffic is inspected by Palo, it will go to its destination.
  - Palo is learning AVS routes and learning spoke routes because of peering
    - You cannot turn off the Palo from learning AVS and spoke routes via BGP
    - The ER GW will always send everything it knows to the Palo
  - Can you create a route table on the NIC that connected to the private/inside subnet and not propagate its routes ?
  - Palos would want to learn the BGP routes from AVS (via NSXT) on a separate route table with propagation enabled, so you can see it as the next hop coming from the ER GW.
  - The NIC that forwards left-right traffic needs to know about BGP which is whatever the ER GW is learning, so it can send traffic there.
  - Next hop on the Palo won't have EGP, instead have a static route towards the AVS inside subnet .1 address. Then the inside subnet will look at its route table and see the BGP learnt routes from the NSXT to route to the destination.
- Why do we need ARS?
  - ARS provides a way to
    - push 0/0 down from Palo to
      - AVS
      - Client/customers connecting via VPN GW
    - Push VPN learnt routes (from the clients/customers) to on-prem, AVS and VNets because AVS acts as a route reflector between VPN GW and ER GW
  - Without ARS, VPN connected clients/customers will only learn the routes of the VNets, not the routes learnt by the ER gateway (ie on-prem and AVS routes)
    - We don't provide transitivity between ER GW and VPN GW connected routes by default
    - VPN clients won't learn the routes learned through ER GW even though they are injected into the VNet, but is not advertised.
  - Without ARS, VPN
    - won't learn the routes learnt by the ER GW (even though they are injected into the VNet, but its not advertised).
      - The ER GW is learning the AVS Routes and the on-prem routes
    - will learn only the connected VNets to the ER GW.
  - Without ARS,
    - the VPN GW will learn the routes of the connected clients (customers) and send it to the VNet
    - The ER GW will learn the route of its connected entities (AVS and on-prem) and sent it to the VNet
    - But they don’t send their learnt routes to each other: the VPN GW does not send its learnt routes to the ER GW. The ER GW does not send its learnt routes to the VPN GW
  - Although the ER GW and the VPN GW are in the same subnet (called GW subnet), they share each other's connected routes with the VNet, but not each other's learnt routes amongst themselves unless there is an ARS with its "enable branch to branch" setting is turned on.
  - ARS does this:
    - Takes routes learned by ER GW send it to VPN GW
    - Takes VPN routes learnt from the customers and send it to ER GW
  - How does ARS connect to VPN GW?
    - Under the hood, once you enable "Branch to Branch" setting on ARS, that will make
      - The Palo have iBGP connectivity to ARS
      - the VPN GW exchange routes though iBGP connectivity.
    - Customers don't see the iBGP connectivity on the portal or anywhere.
  - Palo can advertise 0/0, but it needs something to send 0/0 to the ER GW. Palo cannot be peered directly to the ER GW.
    - How do you generate a default route to send through to an ER GW?
      - You need ARS
  - ARS Acts as a route reflector between the VPN GW and the ER GW. ARS is like another GW besides VPN GW and ER GW
  - The OS on Palo will BGP peer with the two instances of the route server. Palo will originate the 0/0 route and send to ARS. ARS will send it to the ER GW. ER GW will leak it to the AVS and to on-prem
  - Here is the flow for 0/0 traffic:
    - Palo advertises 0/0 to ARS through BGP with next hop ILB (if running in Active-Active)
      - Peer Palo and tell ARS that ILB is the next hop
    - Thus 0/0 (with next hop ILB) gets advertised to AVS and the ER GW
- Source IP preservation: default route required if the VM in AVS needs to see the source IP.
  - Can always do SNAT on the Palo so it looks like its coming from the inside interface on the Palo, but that may not be the only requirement, may need to have source IP preservation. Need to support both: source IP preservation and SNT on the Palo inside interface.
  - Not using SNAT, ARS has an option called "Branch to Branch" which is where Palo can start learning through the ARS. That makes the ER GW send a route to the VPN GW. Now all your clients will be learning all the routes.
  - With SNAT on the on the Palo, you can have a UDR ont eh VPN GW saying if you want to go to AVS, first go to Palo. But that does not solve your clients from learning everything even on-prem.
    - We want the clients to only learn about AVS, not on-prem.
  - There are multiple design options to prevent clients from learning all routes or a simpler design is vWAN. In the next call will go over the multiple design options.

### Pros:
- 
### Cons:
- Complex because clients/customers accessing VPN will know all routes to get to everything on-prem and everything in AVS:
  - With ARS acting as route reflector between ER GW (connecting on-prem and AVS) and VPN GW (connecting clients/customers), the Clients/customers connected to VPN learn all the routing to get to
    - All on-prem
      - Not necessary for clients/customers to learn on-prem routes unless the VMs are split between AVS and Austin
    - All AVS
      - Fine if Clients/customers only need to get to all AVS
  - To prevent the customer/clients from receiving all on-prem routes, build static tunnels won't work
- Routing is Comple: Even if we allow client/customers to learn all on-prem routes, because the traffic needs to be inspected, we will need a UDR on the GW subnet (same subnet hosts both ER and VPN GW) to force the traffic to go to the ILB before it goes to the VPN GW
  - Otherwise, traffic on AVS <-> ER GW <-> VPN GW <-> client happens without inspection
  - UDR on GW subnet should be:
    - AVS Prefixes --Next Hop--> Palo ILB
      - Nightmare for VPN->AVS traffic as AVS Prefixes will grow over time and UDR has to be exact AVS prefixes. It cannot be one single big prefix, it has to be the exact prefix, otherwise the VPN traffic to AVS will bypass the FW.
    - VPN Prefixes --Next Hop--> Palo ILB
      - Nightmare for AVS-> VPN traffic as VPN Prefixes will grow over time and UDR has to be exact VPN prefixes. It cannot be one single big prefix, it has to be the exact prefix, otherwise the AVS traffic to VPN will bypass the FW.
- Higher cost



## Transit/Spoke VNet for VPN:
### Diagram:
![image](https://github.com/user-attachments/assets/fb65e9bb-e673-45a5-8900-febee2965ad7)

### Solution Design:
- VPN Connections are terminated in a VPN GW in a separate VNet (not separate subnet).
- The VNet hosting the VPN GW will have:
  - Route Server
  - Palo
- You need the ARS in the HubVNet to advertise 0/0 down to AVS and on-prem
- VPN GW in a separate VNet provides control over what the VPN GW is learning
  - The Palos in the Hub VNet will BGP peer with the ARS in the Spoke VPN VNet and send only those prefixes that you want the clients/customers to know.
  - The ARS in the Spoke VPN VNet also has a BGP peering with the VPN GW and sends the routes learnt above to the clients/customers.
- From the perspective of the ER GW in the Hub VNet, you can't have ARS (in the VPN Net) as the next hop, nor can you have the VPN GW as the next hop. We need something to do next hop:
  - We need something to do next hop because:
    - AVS will know about VPN because the Palo (in the Hub VNet) through its BGP peering with the ARS (in the Hub VNet) is sending 0/0 - VPN routes (can be summary or 1by 1)
    - AVS -> VPN client traffic flow: AVS -> ER GW -> ILB -> a Palo in the Hub VNet. The Palo in the Hub VNet needs a next hop.
      - The Palo in the Hub VNet has learnt the VPN routes through its BGP peering with the ARS in the VPN Subnet.
      - In the OS of PAN, you will see the VPN GW as the next hop, but from Azure's perspective, you cannot have:
        - next hop to ARS in the VPN VNet. ARS is just a route reflector.
        - Next hop to VPN GW IP. A UDR cannot point to the VPN GW.
          - VPN GW does not have an IP. It just builds tunnels. You cannot point to the VPN GW. Its not like a NVA.
          - One option is to set "use a remote GW" on the peering between the Hub and the Spoke/VPN peering connection.
            - Because we have Gateways in the Hub and the Spoke VNets, you cannot set "use remote GW" on the peering between the Hub and Spoke.
  - That something to point the return traffic is the NVA/PAN in the VPN VNet.
    - Can use Az FW if you don't want to worry about deploying HA pairs and use Az FW more for routing than inspecting.
    - Or can be any flavor of NVA like PAN
  - On the egress interface of the Palo in the Hub VNet, you need a UDR that says:
    - If you want to go to the VPN summary routes, next hop go through the FW in the VPN VNet.
    - UDR: on palo egress send to AzFW back to client
    - The FW in the VPN VNet is already leaning the VPN routes from the VPN GW
  - Return traffic from the client/customers hits VPN GW:
    - The VPN GW is learning (through the ARS in the VPN VNet) that next hop for AVS is the Palo in the Hub VNet because Palo in the Hub VNet is advertising AVS routes to the ARS in the VPN VNet
    - But you cannot point the return traffic hitting the VPN GW to the ARS (in the VPN VNet) to Palos in the hub. Because return traffic is going through the FW in the VPN VNet
    - Hence you need a UDR on the VPN Gateway subnet: AVS ---> Az FW
      - UDR says to go to AVS (ie AVS summary routes), go to the FW in the VPN VNet first.
      - The FW in the VPN VNet is already leaning the routes to get to AVS because the ARS in the VPN VNet is advertising that the next hop is Palo (in the Hub VNet). Hence the traffic goes to the Palo ILB. Then to AVS.


### Pros:
- More control over the routes received by the clients/customers

### Cons:
- Complex routing setup
- Requires a duplication of NVA/Palo & ARS in the VPN Subnet

## vWAN based Solution:
### Diagram:
![image](https://github.com/user-attachments/assets/5e28d838-dd11-46c6-9482-6562cca01080)

### Solution Description:
- Needs Palo inside the VHub
  - Today you cannot have Palo in a spoke
    - If Palo were in a spoke, you can't transit from ER GW to VPN GW via Palo in the spoke in vWAN
  - Requires Palo SaaS licensing different from Palo on VM licensing
    - Not all features of Palo on VM are supported in the Palo SaaS license.
- Simpler design because you don't need UDRs by default:
  - Traffic: AVS -> to anywhere (say Future spokes) that need to be inspected don't need a UDR.
- The issue of the VPN clients/customers learning all routes in AVS and on-prem can be resolved with routeMaps in vWAN.
  - With route Maps, the connection properties of a specific client's VPN tunnel, can specify to drop the on-prem routes and only leave AVS routes.
  - RouteMap option to drop routes on a VPN tunnel does not exist in Hub and Spoke. The RouteMap option is only available in vWAN.
  - With Hub and Spoke the only option to control the on-prem and AVS routes advertised to clients/customers was to setup a Transit VPN GW VNet (as shown above) to control routes being sent by the Palo in the hub VNet (above) to the ARS in the VPN VNet to send to the VPN GW and clients/customers as shown above.
  - The ER GW not being able to manage the network segments can be challenging with routing. If we can manage that from the Palo side, leveraging vWAN and vHUB that would still not make sense.



### Pros:
- Makes the design easier

### Cons:
- Requires Palo SaaS licensing different from Palo on VM licensing
  - Not all features of Palo on VM are supported in the Palo SaaS license.
  - 



# Selection Criteria for End-State:
- Its not favorable for the client to learn all the on-prem route which introduces complexity, but can we compromise and reduce complexity by letting the clients know about the on-prem routes?
- The return traffic from the clients is going to Palo, so you can drop connections there as option.
  - Client Traffic -> VPN GW in Hub -> ILB -> Palo -> ER GW -> On-prem. So you can drop in the Palos if destined on-prem.
- Per Andrew: best solution option is to eliminate the Azure VPN GW based solution and ternminate the VPN tunnels off Palo.
  - This allows for manipulate the traffic because there is overlapping client network tunnels that Palo can handle, but Azure VPN GW has some limitations.
  - Can do NAT on Azure VPN GW, but you can setup the Palos such the customer doesn't have to adjust their side.
- The way Eagle is set up today with ARS and the VPNs will work if executed properly.
- Make sure to drop 0/0 onprem
  - because 0/0 will leak via 2 paths:
    - ER GW -> MSEE -> OnPrem
    - ER GW -> AVS DMSEE -> MSEE -> OnPrem
  - Whenever we peer on the ER, we'll apply route maps (on on-prem routers) for inbound and outbound.
  - The default route
    - will not go: on-prem -> hub in Azure
    - Will go: ER GW in Azure -> on-prem
- Overlapping IP considerations:
  - Some apps have overlapping IP addresses. P21 does not have overlapping IP addresses.
  - Customer A,B and C could all have 192.168.1.0 on their LAN. Depending on the environment, the source subnet is different. Epicor does policy based route to guide traffic based off the source subnet and detination networks.
  - In some cases, we SNAT for the inside VM. Comming from the customer, across the VPN, as those packets are coming in, we source NAT that to un-used IPs. Then the VMs behind would be able to route to those IPs. 
    -  P21 is NATing client subnet. P21 doesn't have overlapping client subnets
    -  You can advertise those NATed IPs from the FW when using a route server using a null route. That creates the BGP advertisement towards ARS. ARS will inject that route into the underlying Azure route tables. The Palos VPN tunnels which are route based injects them into Palo's Route table. 
    -  Walking through the opposite direction to the point above, Palo is advertising a route for a VPN. That route is leant onto the ARS which injects into an Azure Route Table that the VPN GW subnet is in.
    -  Return traffic from AVS -> ER GW - (next hop) -> Palo because of those NATed IPs that Palo is sending through ARS will cause the traffic to to by default to Palo. Palo reverse NATs it and sends it back to the client tunnel.

# Limits:
- ARS can accept 1000 max prefixes from each peer. There are 10K tunnels right now.
  - May require route summary. Should not be a problem since using private IP space.
  - 
