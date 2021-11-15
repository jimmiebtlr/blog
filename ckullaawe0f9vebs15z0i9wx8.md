## K8s Basics: Getting started with Terraform and GKE

Terraform is a technology for managing infrastructure as code.  Many projects would benefit greatly from using it to manage infrastrucutre.

GKE is googles kubernetes offering, and has some great features and offerings to make your experience with kubernetes more amazing.

In this article I'll walk you through some of the options you may want to consider, help you avoid pitfalls, and present my preference for a nice general purpose config.

Feel free to  [skip to the code](#general-purpose-config) , and go back through the article for any arguments that you're curious about.

For tips on writing better cloud native services check out my other article [9 tips for writing better micro-services](https://www.jimmiebtlr.com/9-tips-for-writing-better-micro-services).

## Cluster

We'll start with a minimal config, and add to it.

A basic config will look something like this (the next thing we cover is location which is required).

```
resource "google_container_cluster" "clustername" {
  name     = "clustername"
}
```

### Location

Location is a required field, and an important one to discuss.  This field controls both what regions/zones your cluster runs in, but also whether it runs as a "zonal" or "regional" cluster.

Zonal clusters run in a single availability zone, whereas regional clusters run in multiple but require more instances of every pod (one per availability zone).

If you have enough traffic to make use of multiple instances of each pod, regional will be a great option.  If you need to do everything you can to minimize downtime, you should go with a regional cluster.

If your traffic is minimal, and minimizing cost of resources is important, go with a zonal config.

Zonal 
```
resource "google_container_cluster" "clustername" {
  ...
  location = "us-central1-a"
}
```

Regional 
```
resource "google_container_cluster" "clustername" {
  ...
  location = "us-central1"

  // Optional control over which zones your regional cluster runs in
  node_locations = ["us-central1-a","us-central1-b"]
}
```


### Networking Mode - **IMPORTANT** 

Perhaps the biggest pitfall in setting up your cluster using terraform. 

Terraform managed clusters default to `routes` based clustering, wheras google defaults to `VPC_NATIVE`.    In order to work with private Cloud SQL you'll need VPC_NATIVE. 

More info on  vpc native clusters  [here](https://cloud.google.com/kubernetes-engine/docs/concepts/alias-ips) .


```
resource "google_container_cluster" "clustername" {
  ...
  networking_mode = "VPC_NATIVE"

  // Networking mode VPC_NATIVE also requires an ip allocation policy
  ip_allocation_policy {
    cluster_ipv4_cidr_block  = "/16"
    services_ipv4_cidr_block = "/22"
  }
}
```


### Release channels

Release channels allows you to have google manage your kubernetes update cycle.

Stable is likely good enough for most production workloads, unless you have a specific need for something else.

```
resource "google_container_cluster" "clustername" {
  ...

  release_channel {
    channel = "STABLE"
  }
}
```


### Workload Identity Config

I need to review this,  I haven't been able to get it working properly quite yet.

This option though allows you to specify credentials in a way that new credentials will be generated and used each time a pod is spun up, yet with correct permissions to run what it needs.


### Timeouts

Clusters can take some time to create and update.  To avoid running into these limits in terraform, let's increase the timeouts.  

Without these, there is a higher chance google takes long enough that terraform thinks the operation failed and cancels it.

```
resource "google_container_cluster" "clustername" {
  ...

  timeouts {
    create = "20m"
    update = "20m"
  }
}
```

### Shielded Nodes

Shielded nodes increases the security of your cluster by verifying nodes that join.  Otherwise it's a bit easier to impersonate nodes in your cluster given the ability to exploit a pod.

```
resource "google_container_cluster" "clustername" {
  ...
  enable_shielded_nodes = "true"
}
```

### Maintance Policy

By default google will run automatic upgrades of your cluster whenever it feels like it.  To change it to start in a specific window, this is how.  Time is in GMT.  There are some other options to choose from in maintenance policy, but this is what I use.
 
```
resource "google_container_cluster" "clustername" {
  ...
  maintenance_policy {
    daily_maintenance_window {
      start_time = "03:00"
    }
  }
}
```


### Resource Usage 

Gather data on resource usage within the cluster.  Good for determining what containers are costing what amount.

```
resource "google_container_cluster" "clustername" {
  ...
  resource_usage_export_config {
    enable_network_egress_metering = false
    enable_resource_consumption_metering = true

    bigquery_destination {
      dataset_id = "cluster_resource_usage"
    }
  }
}
```


### Initial Node Count and Remove Default Node Pool - **IMPORTANT**

Node pools specified in the cluster resource require cluster to be destroyed/recreated in order to change them.  For this reason I strongly recommend using a separate node pool config.  This is what's required for the cluster config in order to enable that.

Clusters spun up by terraform have to have a node pool to start. We'll tell terraform to auto destroy that node pool, and have at most one node in it while creating.


```
resource "google_container_cluster" "clustername" {
  ...

  initial_node_count       = 1
  remove_default_node_pool = true
}
```


### Other options

Some other options on the cluster object that I generally leave at defaults, but you may be interested in.

#### enable_autopilot

Autopilot mode gives a more hands off experience for GKE at the expense of less flexibility.  

#### network

Specify what network your cluster runs in.  I think default is often good enough, but for more advanced use cases you may want to use another network.

#### vertical_pod_autoscaling

Still in beta, but sounds like a very nice feature.  This will handle tuning cpu/mem requests for you.  

#### Istio config

Note, google seems to discourage using this option.  The suggest using anthos if you want service mesh features supported by them.  Further the just released a competitor to istio that I imagine will be better supported long term.



## Node Pool

Node pools define what VM's are used to run our kubernetes cluster.  For the rest of the article we'll be covering the configuration of this resource instead of the cluster's config.


### Location (basically the same as cluster)

Zones/Regions should be the same as cluster.

Zonal 
```
resource "google_container_node_pool" "nodepoolname" {
  ...
  location = "us-central1-a"
}
```

Regional 
```
resource "google_container_node_pool" "nodepoolname" {
  ...
  location = "us-central1"

  // Optional control over which zones your regional cluster runs in
  node_locations = ["us-central1-a","us-central1-b"]
}
```


### Auto-scaling

Auto-scaling is a very important feature allowing your cluster to grow and shrink with your traffic.

I'd suggest having a max node of maybe double what you may need at peak.  Basically you want some flexibility and ability to scale, but you don't want a bug making taking up massive CPU to cost massive amounts since your cluster scaled to something huge.


```
resource "google_container_node_pool" "nodepoolname" {
  ...

  initial_node_count = 1
  autoscaling {
    min_node_count = 1
    max_node_count = 12
  }
}
```

### Node Config - Machine Type

What type of VM your node pool uses.  At a rough estimate, I generally target having a low number of nodes, but never less than (number of zones your running in).  

Larger nodes will be easier to efficiently make use of, but if your not using all the resources they provide it'll be a waste.

```
resource "google_container_node_pool" "nodepoolname" {
  ...

  node_config {
    ...

    machine_type = "e2-medium"
  }
}
```

Machine type docs  [here](https://cloud.google.com/compute/docs/machine-types) .


### Node Config - Preemptible

Preemptible VM's are a cheap alternative to regular VM's with the huge caveat that they may be shut down at any point (hopefully with 60s or so warning).

They can save alot of money, but they downside means you'll probably want them only for secondary data processing that isn't directly serving user requests, can shutdown safely and quickly, and/or be restarted/recovered easily.

```
resource "google_container_node_pool" "nodepoolname" {
  ...

  node_config {
    ...

    preemptible = "true"
  }
}
```

### Node Config - Workload metadata config 

Same as workload identity from cluster config, I need to experiment more with this.

Useful for helping secure your cluster more.


### Node Config - OAuth scopes

Some permissions you'll need for your node pool.

```
resource "google_container_node_pool" "nodepoolname" {
  ...

  node_config {
    ...

    oauth_scopes = [
      "https://www.googleapis.com/auth/logging.write",
      "https://www.googleapis.com/auth/monitoring",
      "https://www.googleapis.com/auth/cloud-platform",
    ]
  }
}
```

### Node Config - Metadata - disable-legacy-endpoints

Default value set by GKE.  Terraform will try to unset it though if you haven't specified it.

```
resource "google_container_node_pool" "nodepoolname" {
  ...

  node_config {
    metadata = {
      disable-legacy-endpoints = true
    }
  }
}
```

### Management - Auto repair

Should google recover bad nodes for you?

```
resource "google_container_node_pool" "nodepoolname" {
  ....
  management {
    auto_repair  = true
  }
}
```

### Management - Auto upgrade

Should node pools auto upgrade?

```
resource "google_container_node_pool" "nodepoolname" {
  ....
  management {
    ....
    auto_upgrade = true
  }
}
```


### Timeouts

Extra time for terraform to apply changes to your node pool.  They can take awhile to run.

```
resource "google_container_node_pool" "nodepoolname" {
  ...

  timeouts {
    create = "20m"
    update = "20m"
  }
}
```

### Other Node pool options

#### guest accelerators

Allows attaching gpu's to your nodes.




## General Purpose Config

This is my take a general purpose config, that should be useful in many cases. But read through and find what changes make sense for your use case.



Find the full example at 

%[https://github.com/jimmiebtlr/blog_code/tree/main/getting_started_kubernetes_terraform_gke]


Just the cluster and node pool configs

```
resource "google_container_cluster" "cluster" {
  name     = "cluster"
  location = "us-central1"

  node_locations = ["us-central1-a", "us-central1-b"]

  release_channel {
    channel = "STABLE"
  }

  enable_shielded_nodes = "true"

  workload_identity_config {
    identity_namespace = "${var.project}.svc.id.goog"
  }

  resource_usage_export_config {
    enable_network_egress_metering = true
    enable_resource_consumption_metering = true

    bigquery_destination {
      dataset_id = google_bigquery_dataset.cluster_resource_export.dataset_id
    }
  }

  networking_mode = "VPC_NATIVE"
  ip_allocation_policy {
    cluster_ipv4_cidr_block  = "/16"
    services_ipv4_cidr_block = "/22"
  }

  maintenance_policy {
    daily_maintenance_window {
      start_time = "03:00"
    }
  }

  initial_node_count       = 1
  remove_default_node_pool = true

  timeouts {
    create = "20m"
    update = "20m"
  }
}

resource "google_container_node_pool" "primary" {
  name     = "primary"

  location = "us-central1"
  node_locations = ["us-central1-a", "us-central1-b"]

  cluster  = google_container_cluster.cluster.name

  initial_node_count = 1
  autoscaling {
    min_node_count = 1
    max_node_count = 3
  }

  node_config {
    preemptible  = false
    machine_type = "e2-medium"

    workload_metadata_config {
      node_metadata = "GKE_METADATA_SERVER"
    }

    metadata = {
      disable-legacy-endpoints = true
    }

    oauth_scopes = [
      "https://www.googleapis.com/auth/logging.write",
      "https://www.googleapis.com/auth/monitoring",
      "https://www.googleapis.com/auth/cloud-platform",
    ]
  }

  management {
    auto_repair  = true
    auto_upgrade = true
  }

  timeouts {
    create = "20m"
    update = "20m"
  }
}
```


## Conclusion

GKE is a powerful, full featured and mature offering for kubernetes.  With lots of options though it is even more important to manage your config with infrastructure as code.

Terraform is great for managing GKE, but there are a few pitfalls to be aware of (including some defaults that are different in terraform vs gke).  It can make it much easier to version control your config, test it in a dev setting, and otherwise be more sure the complexity of configuring your cluster can be tested before YOLO'ing it in your production environment.

Let me know what I've missed, what you disagree with, what was unclear or anything you feel like commenting on below! Your feedback helps me improve.
