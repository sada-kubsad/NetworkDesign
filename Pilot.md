# Table of Contents
<details>
<summary>POC</summary>

- [Phase 1: Extend Layer 2](#phase-1-extend-layer-2)
  - [Solution Description](#)
  - [Diagram](#)
  - [Steps](#)
  - [Success Criteria](#)

- [Phase 2: Layer 3 Cutover (Internet and VPN Tunnels), one at a time from on-prem to Azure](#phase-2-layer-3-cutover-internet-and-vpn-tunnels-one-at-a-time-from-on-prem-to-azure)
  - [Solution Description](#)
  - [Diagram](#)
  - [Steps](#)
  - [Success Criteria](#)

- [Phase 3: Move Default GW to AVS](#phase-3-move-default-gw-to-avs)
  - [Solution Description](#)
  - [Diagram](#)
  - [Steps](#)
  - [Success Criteria](#)

- [Phase 4: Move Edge capabilities to Azure](#phase-4-move-edge-capabilities-to-azure)
  - [Solution Description](#)
  - [Diagram](#)
  - [Steps](#)
  - [Success Criteria](#)

</details>

# Pilot/POC:

## Goal:
- Prove that the connectivity solution works
- 
## Phase 1: Extend Layer 2, setup HCX and move servers to AVS
### Solution Description:
- Use a development AVS environment in Austin
- Setup VPN from on-prem to Azure (similating ER)
  - Required because connection to AVS is only via ER, AVS cannot VPN connect to on-prem
    - AVS can do policy based VPN connection to on-prem. That policy based VPN can only terminate in AVS on T1 routers, not T0 routers. 
  - VPN for Testing only. Not ER because ER ports have a minimum 12 months commitment.
    - Latency may not be as good as ER, but is for testing. 
  - Leverage VPN gateway in Azure Hub VNet
  - Azure VPN gateway has to be in Active/Active mode for AVS.
  - This circuit is separate and dedicated for this project. 
- Setup ER GW in VNet
  - Required to connect to AVS environment
- Setup ARS in VNet:
  - Acts as route reflector between VPN GW and ER GW 
- HCX considerations:  
  - HCX is supported over ER, VPN and SD WAN
  - HCX version 10 can turn off HCX encryption when we create HCX tunnels over another VPN IPSec tunnel that is already encrypted. 
- Test VMs move from Austin to AVS, while networking remains on-prem via on-prem GW
- Routing
  - All traffic still flows back through on-prem
  - Internet Traffic
    - Internet egress continue to remain on-prem
    - AVS environment can go out to the internet using Microsoft's direct SNAT  
    - AVS will route Internet traffic through on-prem: AVS -> DMSEE -> ER GW -> On-Prem -> internet
  - VPN Traffic 
- On-Prem will retain all the network equipment: core switches, next layer switches, routes, FW, Internet WAN circuits etc.
### Diagram:
![image](https://github.com/user-attachments/assets/bd25112c-f64b-4d66-981f-f2c0c4defa1b)
### Steps:
1. Setup VPN from Austin/on-prem to Hub VNet in Azure
2. Setup ER GW in Azure Hub
3. Setup AVS environment
4. Link AVS to ER GW created in Azure from step 2
5. Setup ARS in Azure
6. Setup HCX
7. Setup a test application: NGINX over port 80
8. Move/VMotion a test VM from Austin to AVS
### Success Criteria:
- VM is moved fron on-prem to AVS.
- Application connectivity continues to work over Layer 2 extention after VM is migrated to AVS
- Evaluate how Layer 2 stretch works

## Phase 2: Layer 3 Cutover (Internet and VPN Tunnels), one at a time from on-prem to Azure
### Solution Description:
- Add Palos and ILB to the hub in Azure
  - Required to:
    - terminate VPNs
    - Only an NVA (like Palo) can inject routes (default route) into Azure.
    
- Setup client VPN
  - Client VPN to on-prem first to simulate existing environment
    - NGINX on port 80 could be a test
- Setup MON in HCX
  - In preparation for cutover which happens during maintenance window.

- Routing considerations:
  - NVA --(routing info)--> ARS --> AVS, on-prem
  - 
- During a maintenance window: 
  - Move the VPN Tunnels, one at a time from on-prem to Azure
    - Customer changes their VPN peer IP to Azure 
      - The VPN Tunnel will be to a different IP to the Palos in the Hub
    - Make MON configuration changes to route internet and VPN traffic through Azure 
    
    
    - - Can't have the same encryption domain in 2 separate VPNs.
  - Internet connectivity
### Diagram:
![image](https://github.com/user-attachments/assets/e9047f19-f370-45b5-8fe0-944658134e02)
### Steps:
1. Setup Palos and ILB in Azure
2. Setup a simulated client VPN to on-prem
3. Declare a maintenance window
- After a customer's VPN tunnel cuts over from on-prem to Azure, they should have routes to get to AVS.
### Success Criteria:
- Validate VPN cut over successfully from on-prem to Azure/Palos
- Verify application connectivity to port 80 on the VM in AVS  
- After a customer's VPN tunnel cuts over from on-prem to Azure, they should have routes to get to AVS. And AVS should have a single route to get back to the customer's VPN tunnel.


## Phase 3: Move Default GW to AVS:
### Solution Description:
- Moves some traffic by moving the default GW to AVS, but internet traffic flows through on-prem
- When you cut over default GW to AVS, you are cutting over everything. This will take a long time because of the VPN tunnels.
  - Cannot change the VPN peer IPs for 1000s of customers
### Diagram:
- 
### Steps:
- When you cut over default GW to AVS, you are cutting over everything.

### Success Criteria:
-

## Phase 4: Move Edge capabilities to Azure:
### Solution Description:
- Move VPN Traffic one at a time from
### Diagram:
-
### Steps:
- 
### Success Criteria:
- 

# Next Steps:
- Currently In phase 2, are cutting over just VPN, not internet. Need confirmation (from Mays/Broadcom) that VPN and Internet cut over at different times. 
- Build out phases 3 and 4.
