# Terraform Interfaces
In software engineering we have a major problem where we can have two or more _things_ that are related, and need to 
enforce a common API between them and we don't want to strongly couple them together. We have solved this by creating 
interfaces. In Terraform, we often have the same problem. We have two or more _things_, for example, resources for 
the kubernetes control plane, node pools, and helm provisioning charts (let's use datadog, external dns, and istio as 
examples of some helm charts that you might want to capture in IAC). In this example, we have node pools that need to 
know about the k8s cluster and a collection of helm charts that need to know how to connect to the k8s cluster. 

## The Interface Approach
We'll create a hypothetical project with the following structure

```terraform
─── terraform
    ├── k8s
    │   ├── control-plane
    |   |   ├── interface
    |   |   |   └── interface.tf
    │   │   ├── main.tf
    │   │   └── outputs.tf
    │   └── node-pools
    │       └── main.tf
    └── helm
        ├── datadog
        │   └── main.tf
        ├── dns
        │   └── main.tf
        └── istio
            └── main.tf
```

The interface file will look like the following (this is for an Azure storage account as the backend, but a similar 
approach can be done with AWS S3):

```terraform
data "terraform_remote_state" "ad" {
  backend = "azurerm"

  config = {
    resource_group_name  = "terraform"
    storage_account_name = "my-terraform-state"
    container_name       = "tfstate"
    key                  = "k8s-control-plane.tfstate"
  }
}

output "aks_resource_group_name" {
  value = data.terraform_remote_state.control-plane.outputs.aks_resource_group_name
}

output "kubernetes_cluster_id" {
  value = data.terraform_remote_state.control-plane.outputs.kubernetes_cluster_id
}

output "kubernetes_cluster_name" {
  value = data.terraform_remote_state.control-plane.outputs.kubernetes_cluster_name
}

output "node_resource_group_name" {
  value = data.terraform_remote_state.control-plane.outputs.node_resource_group_name
}

output "kubernetes_host" {
  value = data.terraform_remote_state.control-plane.outputs.kubernetes_host
}

output "vnet_subnet_id" {
  value = data.terraform_remote_state.control-plane.outputs.vnet_subnet_id
}
```

We can implement this interface module like:

```terraform
module "k8s-control-plane" {
  source = "../k8s/interfaces/k8s-control-plane"
}

locals {
    aks_resource_group_name  = module.k8s-control-plane.aks_resource_group_name
    kubernetes_cluster_id    = module.k8s-control-plane.kubernetes_cluster_id
    kubernetes_cluster_name  = module.k8s-control-plane.kubernetes_cluster_name
    node_resource_group_name = module.k8s-control-plane.node_resource_group_name
    kubernetes_host          = module.k8s-control-plane.kubernetes_host
    vnet_subnet_id           = module.k8s-control-plane.vnet_subnet_id
}
```

Now, we've in effect published "public" methods that can be consumed by other projects. Node pools and helm charts can 
pull necessary information from the control plane without having to know the implementation details of the control plane, 
nor its outputs. This creates several benefits:
1) By not having any hard dependencies from the children to the control plane, we can decrease the blast radius of 
    accidental changes by children to the control plane.  
2) Terraform works better when the scope of changes are smaller. It runs faster, plans are easier to parse, and it can 
    be parallelized more easily.
3) By pulling out the outputs from the state file it ensures that no one needs to understand the output data structure 
    of the parent project except the interface module. Furthermore, by adding the interface module to a project it 
    allows IDE autocomplete methods to work as intended and it ensures that the happy path is followed.  

This allows you to break up monolithic patterns into a much easier to sustain and more maintainable pattern. While this 
approach might not be especially useful for people that run Terraform locally, this approach is particularly useful 
in conjunction with [TACOS](https://itnext.io/spice-up-your-infrastructure-as-code-with-tacos-1a9c179e0783). With the 
interface approach we increase the complexity of deploys (by having many more environments with their own workspaces) 
but we can pawn off that complexity to automation. The trade off, imo, is a good one for engineers. 
