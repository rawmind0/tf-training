# Rancher2 terraform provider


## Introduction

https://www.terraform.io/intro/index.html

### What is Terraform?

Terraform is a tool for building, changing, and versioning infrastructure safely and efficiently. Terraform can manage existing and popular service providers as well as custom in-house solutions.

Configuration files describe to Terraform the components needed to run a single application or your entire datacenter. Terraform generates an execution plan describing what it will do to reach the desired state, and then executes it to build the described infrastructure. As the configuration changes, Terraform is able to determine what changed and create incremental execution plans which can be applied.

The infrastructure Terraform can manage includes low-level components such as compute instances, storage, and networking, as well as high-level components such as DNS entries, SaaS features, etc.

The key features of Terraform are:

* Infrastructure as Code
Infrastructure is described using a high-level configuration syntax. This allows a blueprint of your datacenter to be versioned and treated as you would any other code. Additionally, infrastructure can be shared and re-used.

* Execution Plans
Terraform has a "planning" step where it generates an execution plan. The execution plan shows what Terraform will do when you call apply. This lets you avoid any surprises when Terraform manipulates infrastructure.

* Resource Graph
Terraform builds a graph of all your resources, and parallelizes the creation and modification of any non-dependent resources. Because of this, Terraform builds infrastructure as efficiently as possible, and operators get insight into dependencies in their infrastructure.

* Change Automation
Complex changesets can be applied to your infrastructure with minimal human interaction. With the previously mentioned execution plan and resource graph, you know exactly what Terraform will change and in what order, avoiding many possible human errors.

### Install Terraform

https://www.terraform.io/downloads.html
https://releases.hashicorp.com/terraform/0.12.26/terraform_0.12.26_linux_amd64.zip

### Terraform configuration

https://www.terraform.io/docs/configuration/index.html

### Writing custom provider

https://learn.hashicorp.com/terraform/development/writing-custom-terraform-providers

## Rancher2 terraform provider

The Rancher2 terraform provider was created by us and it was promoted to be officially supported by hashicorp. Terraform should download and local install the provider if needed.

The provider is not inteded to install Rancher2, it's intended to configure/manage rancher resources once Rancher2 is up and running.

### Introduction

Rancher2 tf provider repo and docs are hosted on Hashicorp:

* Source code: https://github.com/terraform-providers/terraform-provider-rancher2
* Docs: https://www.terraform.io/docs/providers/rancher2/index.html

### Configuration

The provider can be configured in 2 modes:

* Admin: this is the default mode, intended to manage rancher2 resources. It should be configured with the api_url of the Rancher server and API credentials, token_key or access_key and secret_key.

```
provider "rancher2" {
  api_url   = "https://rancher.my-domain.com"
  token_key = "TOKEN"
}
```

* Bootstrap: this mode is intended to bootstrap a rancher2 system. It is enabled if bootstrap = true. In this mode, token_key or access_key and secret_key can not be provided.

```
provider "rancher2" {
  api_url   = "https://rancher.my-domain.com"
  bootstrap = true
}
```

* Both modes can be configured on the same tf file, configuring provider alias on data source and resources. 

```
# Provider config for bootstrap
provider "rancher2" {
  alias = "bootstrap"

  api_url   = "https://rancher.my-domain.com"
  bootstrap = true
}

# Create a new rancher2_bootstrap using bootstrap provider config
resource "rancher2_bootstrap" "admin" {
  provider = "rancher2.bootstrap"

  password = "blahblah"
  telemetry = true
}

# Provider config for admin
provider "rancher2" {
  alias = "admin"

  api_url = rancher2_bootstrap.admin.url
  token_key = rancher2_bootstrap.admin.token
  insecure = true
}

# Create a new rancher2 resource using admin provider config
resource "rancher2_catalog" "foo" {
  provider = "rancher2.admin"

  name = "test"
  url = "http://foo.com:8080"
}
```

More info at https://www.terraform.io/docs/providers/rancher2/index.html#example-usage

### Resources

Users can describe provider objects using resources at tf file. Terraform provider is in charge to create/maintain them when applied `terraform apply`. Every single time that terraform is applied, it check current object status (using Rancher API in our provider), saving it at tfstate (Terraform resources state) and comparing it with user defined status at tf file. If there is some drift on the comparison tf provider should update or recreated the object to configure it as user has defined.

Tf resources (and datasource) support 2 kind of fields:
* Argument: Field value can be defined by the user. It could be `required` or `optional`.
* Attribute: Field value can't be defined by the user. It is `computed` and the value is known after tf apply.

Tf resource arguments and attributes can be used by other objects in the form \<resource_type\>.\<resoruce_name\>.\<arg_name\>

The Rancher2 resources supported by the provider are described on the doc site. All resources are managed by provider in admin mode but [rancher2_bootstrap resource](https://www.terraform.io/docs/providers/rancher2/r/bootstrap.html), that it's managed by provider in bootstrap mode.

Example:

A user has a Rancher2 up and running and he wants to create a new catalog at Rancher

* Define tf file. Provider configuration and rancher2_catalog resource are needed.

```
# Provider config
provider "rancher2" {
  api_url = rancher2_bootstrap.admin.url
  token_key = rancher2_bootstrap.admin.token
  insecure = true
}

# Create a new rancher2 resource
resource "rancher2_catalog" "foo" {
  name = "foo"
  url = "http://foo.com:8080"
}
```

* Init provider (required once). Terraform will download required providers. `terraform Init`
* Plan changes (optional). Terraform will plan needed actions, but will NOT execute anything. `terraform plan`
* Apply changes. Terraform will connect to Rancher and will create the catalog. `terraform apply` 

Expected result is to have foo catalog created at Rancher.

Note: Terraform executons has idempotecy. Next executions should just check that the catalog is on the state defined on tf file, updating it if something has changed.

### Datasources

Users can get provider object data using datasources at tf file. Terraform datasources don't create any object on the provider, they are intended to query data. 

Tf datasource arguments and attributes can be used by other objects in the form data.\<resource_type\>.\<resoruce_name\>.\<arg_name\>

The Rancher2 datasources supported by the provider are described on the doc site.

Example:

A user has manually configured a catalog on Rancher. He knows the catalog name but he needs the url

* Define tf file. Provider configuration and rancher2_catalog resource are needed.

```
# Provider config
provider "rancher2" {
  api_url = rancher2_bootstrap.admin.url
  token_key = rancher2_bootstrap.admin.token
  insecure = true
}

# Get rancher catalog data
data "rancher2_catalog" "foo" {
    name = "foo"
}
```

* Init provider (required once). Terraform will download required providers. `terraform Init`
* Plan changes (optional). Terraform will plan needed actions, but will NOT execute anything. `terraform plan`
* Apply changes. Terraform will connect to Rancher and will get the catalog data and add it to tfstate. `terraform apply` 
* Output data (optional). Besides tfstate data terraform can also output concrete data on stdout.

```
output "rancher2_catalog_url" {
  value = data.rancher2_catalog.foo.url
}
```


Expected result is to have foo catalog data (arguments and attributes) at tfstate.

### Dependencies

Tf defines 2 kind of dependencies:

* Implicit: If a resource or datasource is referencing other resource or datasource argument/attribute, it defines a dependency between them. This is important to control the order to create destroy the objects. 

* Explicit: Defining `depends_on` global argument on resource or datasource. `depends_on = [rancher2_catalog.foo]`

Example: 

Node template has an implicit dependency on cloud credential. 
Node pool has an implicit dependencies on node template and cluster. 
Rancher cluster needs an explicit dependency on node_template in order to remove it properly

```
# Create a new rancher2 RKE Cluster 
resource "rancher2_cluster" "foo" {
  depends_on = [rancher2_node_template.node_template]

  name = "foo"
  description = "Foo rancher2 custom cluster"
  kind = "rke"
  rke_config {
    network {
      plugin = "canal"
    }
  }
}
# Create a new rancher2 Cloud Credential
resource "rancher2_cloud_credential" "foo" {
  name = "foo"
  description= "Terraform cloudCredential acceptance test"
  amazonec2_credential_config {
    access_key = "XXXXXXXXXXXXXXXXXXXX"
    secret_key = "XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX"
  }
}
# Create a new rancher2 Node Template
resource "rancher2_node_template" "foo" {
  name = "foo"
  description = "foo test"
  cloud_credential_id = rancher2_cloud_credential.foo.id
  amazonec2_config {
    ami =  "<AMI_ID>"
    region = "<REGION>"
    security_group = ["<AWS_SECURITY_GROUP>"]
    subnet_id = "<SUBNET_ID>"
    vpc_id = "<VPC_ID>"
    zone = "<ZONE>"
  }
}
# Create a new rancher2 Node Pool
resource "rancher2_node_pool" "foo" {
  cluster_id =  rancher2_cluster.foo.id
  name = "foo"
  hostname_prefix =  "foo-cluster-0"
  node_template_id = rancher2_node_template.foo.id
  quantity = 1
  control_plane = true
  etcd = true
  worker = true
}
```
