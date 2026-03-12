I used to get this error whenever I tried to start my kali machine on my system:
`AMD-V is being used by another hypervisor (VERR_SVM_IN_USE).
VirtualBox can't enable the AMD-V extension. Please disable the KVM kernel extension, recompile your kernel and reboot (VERR_SVM_IN_USE).`

The fix for this is to go to the terminal and type:
``
sudo modprobe -r kvm
sudo modprobe -r kvm_amd
``
*Note: You can either use one or both. They do the same thing*

