# Linux Kernel Compilation

In this workshop you will learn how to compile and install new Linux kernel on Ubuntu machine.

## Vagrant commands

1. Provision VM machine:
```bash
vagrant up
```
2. SSH into the machine:
```bash
vagrant ssh
```
3. To pause machine:
```bash
vagrant halt
```
4. To destroy machine:
```bash
vagrant destroy
```

## Kernel Compilation

To download Kernel go to:
- [kernel.org](https://kernel.org/)
- [github.com/torvalds/linux](https://github.com/torvalds/linux)

To check Linux kernel version:
```bash
uname -r
```

By default Ubuntu 22.04.5 LTS VM uses `5.15.0-179-generic` Linux kernel version. In this workshop you have to compile new Linux kernel `5.15.137` version and install it on VM machine.

## Support 

If you find this exercise valuable, consider supporting it on Ko-fi:

<a href='https://ko-fi.com/B0B61Z2ERC' target='_blank'><img height='36' style='border:0px;height:36px;' src='https://storage.ko-fi.com/cdn/kofi6.png?v=6' border='0' alt='Buy Me a Coffee at ko-fi.com' /></a>
