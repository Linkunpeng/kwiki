# DRM (Direct Rendering Manager) 是 Linux 内核中一个至关重要的子系统，它在现代图形栈中扮演着核心角色，负责安全、高效地管理和抽象图形硬件资源

## DRM 是如何工作的

### 内核态：DRM 核心与 DRM 驱动

- DRM 核心 提供了内核中用于管理 GPU 资源的通用框架和 API。
- 当一个 DRM 驱动（如 amdgpu、i915、nouveau 或闭源 nvidia 驱动）被加载并初始化时，
  - 调用 DRM 核心提供的 API（例如 drm_dev_register()）来注册自身，表明它能够管理特定的 GPU 硬件。
  - 驱动程序向 DRM 核心提供一系列回调函数指针，这些函数是 DRM 核心在需要执行显存分配、命令提交、模式设置等操作时会调用的。

### 用户空间：libdrm 与应用程序

- 当应用程序想要与 GPU 交互时，它会通过 libdrm 这个库来完成。
- libdrm 提供了API例如 drmOpen()、drmModeSetCrtc()、drmPrimeFDToHandle()...
- 这些 API 内部会执行与 DRM 驱动通信所需的 ioctl 系统调用。
