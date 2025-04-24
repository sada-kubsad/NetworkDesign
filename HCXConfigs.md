# 
- Separate isolated networks for VMs, hosts and VCenter.
# Vcenter onprem to VCenter in AVS 
- Don't support Linked mode between VCenters
  - Used to manage VCEnters deployed on-prem and on AVS.   
- HCX facilistes the communication between on-prem and Azure for VM Migrations and VMotions
  - IN HCX, when you setup a service mesh it deploys 
    - the interconnect appliances
      - Acts as  hosts to the local VCenter and tunnel the traffic betweeen sites
    - the network extension appliances
