# ZYNQ 7000 LVDS VDMA Framebuffer Driver

Env :
- Platform  : ZYNQ 7000
- IDE       : Petalinux 2019.1
- Kernel    : 4.19.0

Ref code : https://github.com/topic-embedded-products/kernel-module-vdmafb/

## Usage(In Petalinux)

### Creata Petalinux Component

```bash
source <PETALINUX>/settings.sh

cd <Your PetaLinux Project>

petalinux-create -t modules --name  vdmafb --enable
```

### Apply driver code

```bash
cp <THIS REPO>/vdmafb.c  <Your PetaLinux Project>/project-spec/meta-user/recipes-modules/vdmafb/files/vdmafb.c
```

### Add DTS Node

add following code for `project-spec/meta-user/recipes-bsp/device-tree/files/system-user.dtsi`

```c
/ {
    // VDMA fb device
    axi_vdma_vga: axi_vdma_vga@7e000000 {
        compatible = "topic,vdma-fb";
        reg = <0x7e000000 0x10000>;
        dmas = <&axi_vdma_0 0>;
        dma-names = "video";

        num-fstores = <3>; // Xilinx VDMA requires clients to submit exactly the number of frame stores.

        // Change to right Timing for LVDS Screen.
        width = <1920>;
        height = <1080>;

        horizontal-front-porch = <88>; // hfp
        horizontal-back-porch = <128>; // hbp
        horizontal-sync = <44>;        // hs

        vertical-front-porch = <4>;    // vfp
        vertical-back-porch = <36>;    // vbp
        vertical-sync = <5>;           // vs
    };
};
```

### Build

```bash
cd <Your PetaLinux Project>

petalinux-build -c vdmafb
# or
petalinux-build
```

### Test

copy `<Your PetaLinux Project>/build/tmp/sysroots-components/plnx_zynq7/vdmafb/lib/modules/4.19.0-xilinx/extra/vdmafb.ko` or `<ZYNQ 7000 rootfs>/lib/modules/4.19.0-xilinx/extra/vdmafb.ko` to ZYNQ 7000.

```bash
insmod vdmabf.ko
# or
modprobe vdmabf
```


## Application Code

```c
#include <stdio.h>
#include <string.h>
#include <stdlib.h>
#include <fcntl.h>
#include <unistd.h>
#include <linux/fb.h>
#include <sys/mman.h>
#include <sys/ioctl.h>

struct fb_var_screeninfo fb_var;
struct fb_fix_screeninfo fb_fix;

void draw_rect(unsigned int *fbpp, int xi, int yi, int width, int heigh, int color)
{
    int x, y;
    long int location;
    for (y = yi; y < yi + width; y++) {
        for (x = xi; x < xi + heigh; x++) {
            // 计算像素位置
            location = (y * fb_fix.line_length) + (x * (fb_var.bits_per_pixel / 8));
            // 直接通过指针访问并设置颜色（注意这里我们使用了fbpp来简化操作）
            fbpp[(y * fb_var.xres + x)] = color;
            // 注意：上面的计算方式假设了每行像素是紧密排列的，这在大多数情况下是正确的，
            // 但如果屏幕有padding或者使用了特殊的行对齐方式，则可能需要使用fb_fix.line_length来计算。
            // 这里为了简化，我们直接使用了fb_var.xres，这在大多数情况下也是可行的，特别是当fb_var.xres_virtual等于fb_var.xres时。
        }
    }
}

int main(int argc,char *argv[])
{
    int fb = 0;
    long int screensize = 0;
    char *fbp = 0;
    unsigned int *fbpp;


    char *env = NULL;

    short *picture;

    // 打开FrameBuffer设备
    env = argv[1] ? argv[1] :"/dev/fb0";
    fb = open(env,O_RDWR);
    if (fb == -1) {
        perror("Error opening framebuffer device");
        exit(EXIT_FAILURE);
    }
    printf("Success opening framebuffer device %s\n", env);

    // 获取屏幕信息
    if (ioctl(fb, FBIOGET_VSCREENINFO, &fb_var) == -1) {
        perror("Error reading variable information");
        exit(EXIT_FAILURE);
    }
    printf("fb_var.xres=%d/%d\n", fb_var.xres, fb_var.xres_virtual);
    printf("fb_var.yres=%d/%d\n",    fb_var.yres, fb_var.yres_virtual);
    printf("fb_var.bits_per_pixel=%d bit as %d byte\n",    fb_var.bits_per_pixel, fb_var.bits_per_pixel / 8);

    if (ioctl(fb, FBIOGET_FSCREENINFO, &fb_fix) == -1) {
        perror("Error reading fixed information");
        exit(EXIT_FAILURE);
    }
    printf("fb_fix.line_length=%d\n",fb_fix.line_length);
    printf("fb_fix.accel=%d\n",fb_fix.accel);

    // 计算屏幕尺寸
    screensize = fb_var.yres_virtual * fb_fix.line_length;

    // 映射FrameBuffer到用户空间
    fbp = (char*)mmap(0, screensize, PROT_READ | PROT_WRITE, MAP_SHARED, fb, 0);
    if ((int)fbp == -1) {
        perror("Error: failed to map framebuffer device to memory");
        exit(EXIT_FAILURE);
    }

    // 转换为无符号整型指针，方便操作32位颜色值
    fbpp = (unsigned int*)fbp;

    // 绘制一个红色的正方形
    // 假设正方形的左上角在(100, 100)，边长为100像素
    int x, y;
    unsigned int red = 0xFFFF0000; // 32位ARGB颜色，红色
    unsigned int green = 0xFF00FF00; // 32位ARGB颜色，红色
    unsigned int blue = 0xFF0000FF; // 32位ARGB颜色，红色
    unsigned int color[] = {red, green, blue}; // 32位ARGB颜色，红色

    memset(fbp, 0x00, screensize);
#if 1
    for (y = 0; y < 100; y++) {
        for (x = 0; x < 200; x++) {
            // 计算像素位置
            long int location = (y * fb_fix.line_length) + (x * (fb_var.bits_per_pixel / 8));
            // 直接通过指针访问并设置颜色（注意这里我们使用了fbpp来简化操作）
            fbpp[(y * fb_var.xres + x)] = red;
            // 注意：上面的计算方式假设了每行像素是紧密排列的，这在大多数情况下是正确的，
            // 但如果屏幕有padding或者使用了特殊的行对齐方式，则可能需要使用fb_fix.line_length来计算。
            // 这里为了简化，我们直接使用了fb_var.xres，这在大多数情况下也是可行的，特别是当fb_var.xres_virtual等于fb_var.xres时。
        }
    }
#else
    //draw_rect(fbpp, 0, 0, 1920, 1080/3, red);
    draw_rect(fbpp, 0, 100, 0, 1080, green);
    draw_rect(fbpp, 0, 1080/3*2, 1920, 1080/3, blue);
#endif

    // 清理和退出（在实际应用中，你可能希望保持FrameBuffer的映射直到程序结束）
     munmap(fbp, screensize);
     close(fb);

    // 注意：由于我们直接在屏幕上绘制了内容，因此程序结束后这些内容仍然会留在屏幕上，
    // 除非有其他程序（如另一个FrameBuffer程序或图形界面）覆盖了它们。

    return 0;
}

// 注意：上面的代码中的像素位置计算部分做了简化处理。
// 在实际情况下，你应该总是使用fb_fix.line_length来计算每行的字节数，
// 因为屏幕的行可能包含额外的填充字节（padding），以确保每行从某个特定的内存地址开始。
// 但是，对于许多现代屏幕和驱动，这些填充字节可能不存在，或者每行的像素确实是紧密排列的。
```

