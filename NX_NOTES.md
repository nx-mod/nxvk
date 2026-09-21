# nxvk (nx-mod fork)

NVK - Mesa's Vulkan driver for NVIDIA GPUs - running on the Nintendo Switch. Used by
[wii-nx](https://github.com/nx-mod/wii-nx).

Upstream is [PalindromicBreadLoaf/nxvk](https://github.com/PalindromicBreadLoaf/nxvk). This fork pins a
known-good revision, adds CI that builds the packages and every smoke-test app, and carries one fix:

**The window was left unusable after a Vulkan swapchain was destroyed.** Disconnecting does not clear
the buffer slots a swapchain registered, so the next user of the screen - libnx's two-buffer console,
say - was handed a stale slot as soon as its own two were busy, and the queue died
(`LibnxBinderError_NoInit`, surfacing as a fatal `BadGfxDequeueBuffer`). The swapchain now unregisters
its buffers before letting go. Worth upstreaming: it affects any app that hands the screen back.

Prebuilt packages are published on this fork's [releases](https://github.com/nx-mod/nxvk/releases):
the Vulkan and OpenGL portlibs, and the smoke-test NROs.

## How wii-nx consumes it

- Installed as a devkitPro portlib (`make image && make && sudo make install`), giving `libnvk.a`
  and `libnvk_support.a`.
- Linked whole-archive with `-Wl,-u,vk_icdGetInstanceProcAddr`: the driver's dispatch tables are only
  reached by runtime string lookup, so the linker would otherwise drop them.
- Reached through its single exported entry point; [dawn-nx](https://github.com/nx-mod/dawn-nx) resolves
  everything else from there.
- `NVK_I_WANT_A_BROKEN_VULKAN_DRIVER=1` must be set before device creation.
- The newlib gaps the driver expects (`sysconf`, `getrandom`, `posix_memalign`, …) are filled in dawn-nx.

Licensing: NVK is GPL, and it links statically, so anything shipping it is GPL-covered. wii-nx is
GPL-3.0 and its source is public.

It also builds OpenGL 4.5 / ES 3.2 through Zink; wii-nx uses the Vulkan path only.

## Releases

Prebuilt packages are tagged `<upstream version>-nx-mod-v<n>`, the same convention across every nx-mod
library, so a project can pin one line per dependency.

## EGL_KHR_fence_sync on Horizon

`src/egl/drivers/horizon/egl_horizon.c` advertised four extensions
(`KHR_create_context`, `KHR_no_config_context`, `KHR_surfaceless_context`,
`EXT_create_context_robustness`) and its `_eglDriver` had no sync entry points.
Plain GL never notices - none of the fifteen GL smoke tests in `switch/smoke/`
calls `eglCreateSyncKHR` - but Dawn's OpenGL backend refuses a display without
one of the sync extensions:

    BackendGL.cpp: "EGL_KHR_fence_sync or EGL_KHR_reusable_sync must be supported"

so WebGPU-over-Zink reported "No supported adapters" here while the Vulkan path
ran clean. The driver now implements `EGL_KHR_fence_sync` and `EGL_KHR_wait_sync`
over the gallium fence that `st_context_flush` hands back: create flushes and
takes the fence, `eglClientWaitSyncKHR` is `fence_finish` with the caller's
timeout, `eglWaitSyncKHR` is `fence_server_sync` where the driver has it.

Fence rather than reusable: a reusable sync is signalled by the application and
says nothing about the GPU, and callers asking for a sync want to know when the
work landed.
