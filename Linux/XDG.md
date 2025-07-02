# XDG(X Desktop Group)
> XDG 是freedesktop.org的前身，旨在为应用程序提供兼容的Linux桌面环境。

## freedesktop.org
- freedesktop.org为Linux诸多发行版提供了一套统一的桌面环境标准。并维护了许多项目例如dbus,wayland,xorg等等.
- 应用程序根据freedesktop提供的标准进行编程， 以兼容不同的桌面环境例如:
```
XDG_CACHE_HOME=/home/lkp/.cache
XDG_CONFIG_HOME=/home/lkp/.config
XDG_DATA_HOME=/home/lkp/.local/share
XDG_RUNTIME_DIR=/run/user/1000
XDG_SEAT=seat0
XDG_SESSION_CLASS=user
XDG_SESSION_ID=1
XDG_SESSION_TYPE=x11
XDG_VTNR=7
```
