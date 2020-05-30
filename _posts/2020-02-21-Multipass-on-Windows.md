---
layout: post
title:  "Multipass on Windows"
---


I find interesting to use [__Multipass__][mpref]. It is a lightweight tool created by Canonical
to ease development on a personal workstation. Its functions are similar to 
Hashicorp's [__Vagrant__][vagref] but it works only with Ubuntu cloud images. It's interesting because it allows 
a developer to configure a cloud VM image on first launch using the built-in [__cloud-init__][cinitref] provider.  

The cloud-init provider is important because most of the time a cloud infrastructure needs a variety of base images.
To configure them at launch is possible but the process takes time. It is possible to develop a set of preconfigured
base images from which to start, but this involves a significant maintenance effort. Cloud-init offers 
the best tradeoff: the VM launches fast and it can be safely configured on startup. \
Multipass offers a good development and test environment for the cloud-init scripts 
because a developer doesn't need to create a VM on cloud to develop it's configuration script (except maybe
for the final stages of the development/test process).  

Multipass works with a variety of VM hosts. On Windows, I think the most representative are Hyper-V and VirtualBox. 
I prefer to use VirtualBox on Windows, because it offers me more features than Hyper-V. This comes with some
challenges, as I described in the [__technical article__][mpwinref].





[mpwinref]: {% link _articles/Using-Multipass-on-Windows.md %} "Using Multipass on Windows"
[mpref]: https://multipass.run/
[vagref]: https://www.vagrantup.com/
[cinitref]: https://cloudinit.readthedocs.io/en/latest/# "cloud-init documentation"

