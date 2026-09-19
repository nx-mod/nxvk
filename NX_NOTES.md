# nxvk (nx-mod fork)

NVK - Mesa's Vulkan driver for NVIDIA GPUs - running on the Nintendo Switch. Used by
[wii-nx](https://github.com/nx-mod/wii-nx).

**Used unmodified.** This fork exists to pin a known-good revision; upstream is
[PalindromicBreadLoaf/nxvk](https://github.com/PalindromicBreadLoaf/nxvk).

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
