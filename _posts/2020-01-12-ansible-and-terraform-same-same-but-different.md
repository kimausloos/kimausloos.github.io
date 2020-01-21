---
layout: post
title: Ansible & Terraform; same same but different
date: 2020-01-12 12:20 +0100
---

The last coupe of months I dedicated a chunk of my time helping other teams using Ansible as glue and configuration tool for various devices, ranging from network equipment and storage boxes to creating new VMWare vCenter vm's. Since I have been using Ansible for years they appreciated the fact that they could ask me a question and I could give them the commands to use and how (not) to handle the situation. When they discovered I was also using Terraform to set up a whole new environment on AWS they where pretty confused and started asking me why I would use Terraform when Ansible also has all the modules I need.

<br />

### Why Ansible?
Let me start by saying Ansible is awesome! I've been using it since somewhere near the end of 2012 and while in the beginning it had some rough edges they managed to clean it up (mostly) and now it's a very versatile tool with modules for nearly everything. It doesn't really care what you want to do, you can provision a new Linux vm with it but also do software installs (Red Hat uses it for OpenShift for example) or spinning up new vm's in cloud vendors like AWS, Azure, Hetzner, ...

This is a big plus for Ansible; you can do a bit of everything with it and it's a great way to write workflows. It supports hooks so you can execute tasks when another component changed. This way you can, for example, triggering a deploy in Jenkins when it discovers a new release in git. The roles support is also a nice addition, together with [Ansible Galaxy](https://galaxy.ansible.com/) you can download pre-made roles and have your infra configured in no-time. You can also (in theory) run the same playbook 50 times and you should get the same result. There are exceptions, and you can screw this up, but in general you're pretty safe.

When you manage multiple operating systems, templates give you a way to easily configure them all. When using them you can add support for a new os quickly. Testing playbooks and roles is something supported pretty much out of the box and should give you a nice feeling knowing you'll be aware when something breaks.

<br />

### Why Terraform
Terraform is a bit younger and started in 2014. It was one of the first tools I personally remember being built in Go, so it's completely cross-platform and it doesn't matter what os you run it on. It's a real `infrastructure as code` tool and it allows you to write up a whole infrastructure. Since it's start it also has support for providers, ranging from cloud platforms as AWS, gcloud and Azure to DNS hosts (NS1, Akamai) and kubernetes. Community providers are also available so you'll have a rather large choice of services and tools, but still a lot less than Ansible.

In the current version it's also possible to create modules. A module is a (small) unit of infrastructure, like for example a multiple vm node consul cluster with firewall rules. You can re-use the module in multiple places too, so if you have an internal tool and something you deployed for a customer that both use consul you only need to write the code once. The modules can be fetched from git or you can use the [Terraform module registry](https://registry.terraform.io/).

Terraform cares a lot about state and when you execute a plan it will generate a `.tfstate` file. That file contains the detailed state of every resource it created and it's important that you keep the state file and make sure there aren't multiple versions of the same infrastructure state. In the early days when you lost your state file you were basically screwed, but nowadays [you can import resources](https://www.terraform.io/docs/state/import.html) but your mileage still may vary. Having the state saved is nice as you can detect if the real state is out of sync with your last apply, so you know something is up. The state is just a json file by default, but you can store it in S3, consul, an http endpoint, Terraform Cloud, ... Most of the endpoints also support locking the statefile, so you don't run an apply when your colleague is updating the infrastructure.

Since Terraform is all about infrastructure it can also take down a full infrastructure in no time. This is extremely handy when you're creating small proof of concept environments on a cloud provider and want to make sure you won't have extra costs because you forgot to delete a resource. A `terraform destroy` will make sure to delete it all. Or if not possible due to a timeout or something, it will let you know not everything is gone so you can try again. For most people this sounds scary, but it's also a very cool way to make sure you don't end up paying for that Amazon Aurora database you forgot to delete after running that Ansible playbook 6 moths ago.

<br />

### What when you need them both?
You probably get now Ansible is all about running tasks, configuring machines; but also about workflows that touch a bit of everything. Terraform is the tool to make sure you have a complete environment spun up on AWS and also delete it all when you're done with it. But sometimes you want to spin up a whole environment and then run some workflows to configure everything or do deploys.

Well, Ansible has a [Terraform module](https://docs.ansible.com/ansible/latest/modules/terraform_module.html) that allows you to run `terraform apply` while executing an `ansible-playbook`. That way you can start creating a complete infrasturcture and when done (successfully) start doing the rest of the actions needed to make sure everything is done.

On the other hand you can also start Ansible when Terraform is done via the [local-exec](https://www.terraform.io/docs/provisioners/local-exec.html) or [remote-exec](https://www.terraform.io/docs/provisioners/remote-exec.html) provisioner.

It depends on your use-case what approach is best, but it's nice to have multiple options. I used the "Ansible that starts Terraform" approach a couple of times and it worked well. It's a nice solution because you can use the best tool for the job as they both have strong and weak points.

<br />

### Conclusion
You can have it all and should use Terraform to code your whole infrastructure and Ansible to start your custom workflows or configuration of vm's.

It's possible to do everything what Terraform does in Ansible, but just because you can doesn't mean you should. It will be a lot of work to re-create it and you'd have to support everything yourself.

 It's not possible to have Terraform do everything Ansible does, but that's fine as Terraform is a lot quicker and the support for keeping state is a deal breaker. If you want to be able to delete everything and avoid extra costs, Terraform is the way to go.
