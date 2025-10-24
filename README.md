# multitenant-cluster


## Objectives
This validated pattern is designed to support demonstrating and testing emerging multi-tenant networking constructs including the provisioning lifecycle of these assets.
This is a work in progress and should not be considered generically stable 



### Scenario 1
Raw testing of networking constructs: OpenShift and OVN-K have or are working on various enhancements for the networking stack which can help support multi-tenant usecases including:

- User Defined Networks and Cluster User Defined Networks
- BGP integration with OVN-K and Cluster User Defined Networks
- BGP-eVPN integration.
- ...

The objective is to configure all of this as GitOps. This includes the required base cluster configuration. For this a small number of specific tests are required.

#### Simple VPC
1. User can provision a 'VPC' which is a (C)UDN. That construct has an egress IP assigned to it and load balance IP's over the

1. Demonstrating that CUDNs can be peered to a VRF, providing an service provider the flexibility to setup custom routes (north and south) for a tenant using a VRF-Lite pattern
2. Demonstrating that egress IP + CUDN + metallb + egress services can provide the core of a VPC functionality for end users
3. Service provider services: Demonstrate that a service provider can leak services in the cluster to allow a tenant to access a service they run from within a CUDN.

### Scenario 2
I am a service provider providing a cloud experience for my tenants. This IaaS infrastructure allows a simple pattern:


#### Provisioning
1. The sales team at the service provider kick off an onboarding
2. The tenant is registered into the identity provider
3. The tenant provides the key information for their tenancy:
    1. IPv4 CIDR block for the tenancy
    2. number of Public IPv4 addresses required for load balancing and or floating IPs
4. Automation then provisions the required OpenShift components:
  1. IPv4 addresses are checked out of IPAM for the end user
  2. Namespace, CUDN, Egress IP's and Load balance IPs are allocated.
  3. Components are status checked
5. User is furnished with details of the infrastructure.

#### VM provisioning
1. Users can freely provision VMs leveraging the CUDN. (self service, UI and API driven)
2. Users can establish a 'floating IP' self service, UI and API driven)
3. Users can setup load balancing
  - NOTE: Investigate gateway API / routes / L7 capabilities



#### Decommissioning
1. Tenant puts a service request in to delete tenancy
2. Tenancy deletion will occur if no resources attached to the CUDN
3. Failure path of A report of what the tenant needs to delete before tenancy can be deleted.



### Why are the two scenarios combined in the same pattern?
In a service provide differentiation is needed between how self-service workflows for end tenants are provisioned which are ephemeral and react to external stimulus (scenario 2).
Scenario (1) can be viewed as how a service provider would manage their internal infrastructure and/or custom requests which are not possible to be completed with automated workflows.



## Current assumptions in the pattern

1. Built for our lab. Resiliency / customizability will come over time.
2. Single bare metal node deployment.
