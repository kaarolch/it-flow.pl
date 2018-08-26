---

title: "[En] Minikube && Fedora 27 - failed to start after reboot"
date: '2018-08-26T21:43:00.000+02:00'
tags:
- Minikube
- Kubernetes
- HomeLab
---

Install minikube was finished, everything was working fine. Next day command `minikube start --vm-driver kvm2` return error:
````
Starting local Kubernetes v1.10.0 cluster...
Starting VM...
E0826 21:48:31.660132    8562 start.go:174] Error starting host: Error starting stopped host: Error creating  Message='Requested operation is not valid: network 'minikube-net' is not active').

 Retrying.
E0826 21:48:31.660261    8562 start.go:180] Error starting host:  Error starting stopped host: Error creating VM: virError(Code=55, Domain=19, Message='Requested operation is not valid: network 'minikube-net' is not active')

````
Problem: Libvirt was not activated after restart. Simple workaround :

````
sudo virsh net-start minikube-net && sudo virsh net-autostart minikube-net
````
