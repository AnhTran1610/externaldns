# Deploy and use external-dns for Route53 in Kubernetes with Helm and Terraform

## What external-dns does

Basically, **external-dns** is a pod that runs in your Kubernetes cluster, reads your ingresses and services and creates DNS records in a DNS zone manager accessible via API. We use AWS **Route53**, but it can work with all cloud providers, PowerDNS, and so on ! It supports even bind9 and active directory using DNS UPDATE messages, see RFC 2136 for more info. This all sounds very simple, but setting things correctly to fit our use-case took quite a bit of documentation reading.

## Our external-dns use case
### Requirements
I can't make breaking changes to the infrastructure just because I add a DNS stack. Full compatibility with my colleagues' work is a requirement: they did a great work and already deployed a lot of stuff! For instance, we have an alb-ingress-controller in our Kubernetes cluster to create load balancers: I can't decide to change variables used by this piece if I need other values in my DNS stack. Also, I need to come up with a solution that allows piece-by-piece migration: we want to migrate our DNS management deployment by deployment, as per the SRE/devops motto "atomic changes, often". I would also prefer not to redesign half our infrastructure stack just to add my DNS brick. Therefore, I need:
- To deploy external-dns with the Helm Terraform provider within the same Terraform module that deploys my EKS cluster
- A well-maintained Helm chart to deploy external-dns
- To be able to integrate with the alb-ingress-controller
- To let external-dns create DNS records in Route53 but only for specific zones
### IAM and Kubernetes permissions
Given the above requirements, I have to let external-dns create and delete Route53 records.

After digging into external-dns documentation and AWS documentation, I came up with a clean solution:

- Create an IAM Role that allows an external-dns Kubernetes service account to perform an assumeRole using our EKS OIDC provider
- Create a service account for external-dns in Kubernetes that uses the above IAM Role
- Create an IAM policy that allows modifying Route53 records and attach it to the above IAM Role
- Create a Kubernetes cluster role (following external-dns docs) allowing external-dns to read the necessary information about the ingresses and services, and attach it to the service account with a cluster role binding.
- Deploy external-dns with the Helm chart bitnami/external-dns, using the correct parameters.