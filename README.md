## 修复 VMWARE 虚拟机缺少 vmmon 内核模块的问题

```bash
uname -r
```
宿主机版本：Archlinux 内核版本 6.13.2-arch1-1

```bash
vmware --version
```
虚拟机版本：VMware Workstation 17.5.1 build-23298084

```bash
sudo pacman -S base-devel linux-headers
```

编译 vmmon
```bash
# 切换到对应的分支
git pull origin workstation-17.5.1
sudo make
sudo make install
```

直接编译会出问题，修改部分代码

将 `vmnet-only/vmnetInt.h` 中的宏定义
```c
#define dev_lock_list()    read_lock(&dev_base_lock)
#define dev_unlock_list()  read_unlock(&dev_base_lock)
```

修改为如下
```c
#if LINUX_VERSION_CODE >= KERNEL_VERSION(6, 9, 9)
#  define dev_lock_list() rcu_read_lock()
#  define dev_unlock_list() rcu_read_unlock()
#else
#   define dev_lock_list()    read_lock(&dev_base_lock)
#   define dev_unlock_list()  read_unlock(&dev_base_lock)
#endif
```

再重新 make, 就没有问题了。
编译安装完成后，加载 vmmon 和 vmnet 模块
```bash
sudo modprobe vmmon
sudo modprobe vmnet
```

设置开机自动加载
```bash
sudo tee /etc/modules-load.d/vmware.conf <<EOF
vmmon
vmnet
EOF
```

这时候执行就可以看到这个模块了。
```bash
lsmod | grep -E 'vmmon|vmnet'
```

```bash
vmnet                  81920  0
vmmon                 172032  0
```


不知道为什么使用 vmware 自带命令不行。
```bash
sudo vmware-modconfig --console --install-all
```

该服务不叫这个名字了吗？
```bash
[AppLoader] GLib does not have GSettings support.
Failed to stop vmware.service: Unit vmware.service not loaded.
Unable to stop services
```

This repository tracks patches needed to build VMware (Player and
Workstation) host modules against recent kernels. As it focuses on recent
kernels (older ones do not need patching), only vmmon and vmnet modules are
currently handled as the rest has been upstreamed for some time.

Main branch master handles only "infrastructure" files which do not belong
to VMware module sources. Two other branches, "player" and "workstation"
track upstream module sources distributed with Player and Workstation,
respectively. Tags of the form "p${version}" (e.g.  "p12.5.5") and
"w${version}" correspond to clean unpacked sources of modules from
a particular version of Player or Workstation.

From these tags, branches "workstation-${version}" is forked. This branch
tracks changes needed to build the modules against recent kernel versions.
In general, one should always use current branch head for the build. For
versions before 17.0, there are also branches "player-${version}" but as
the module sources have been identical between Workstation and Player for
quite long, there seems to be no need to duplicate the work. Therefore the
"workstation-*" branches should be also used for Player >= 17.0 (and can be
in fact used for older as well). If the situation changes in the future,
Player related branches can be introduced again.

In the past, tags in the form "w${ver}-k${ver}" and "p${ver}-k${kver}" were
also provided to mark the snapshots deemed sufficient to build modules for
Workstation/Player version $ver at the moment of kernel $kver release. This
practice turned to be a bad idea; more often an issue affecting older
kernel versions was discovered later than a fix for newer kernel did not
work with older ones. Unfortunately, misinterpreting these tags often
resulted in building modules from old branch snapshots and reporting issues
that have been addressed long ago. Therefore, starting with kernel 6.0,
these per kernel tags are no longer going to be provided.

At the moment, changes are tested to build against all (vanilla) kernel
releases starting with 4.9.

This repository is provided "as is" with no guarantees. Use the contents on
your own risk.
