使用SurfaceFlinger送图，可以简单的按照如下几个步骤：

创建SurfaceComposerClient   --->具体过程见《构造surfaceComposerClient》
获取要显示的屏幕，这里获取的是主屏SurfaceComposerClient::getInternalDisplayToken
获取屏幕大小 SurfaceComposerClient::getActiveDisplayMode(displayToken, &displayMode);
创建SurfaceControl
设置surface属性；setLayer，setPosition
创建surface
分配并map buffer；surface->lock
向buffer里填充图像数据
推图 surface->unlockAndPost()
源码如下，会在屏幕中间显示一个大小200x200，颜色rgb依次循环的方块。

```cpp
#define LOG_TAG "SurfaceTestDemo"

#include <binder/IPCThreadState.h>
#include <binder/ProcessState.h>
#include <binder/IServiceManager.h>
#include <hardware/gralloc.h>
#include <ui/GraphicBuffer.h>
#include <utils/Log.h>
#include <gui/Surface.h>
#include <gui/SurfaceControl.h>
#include <android/gui/ISurfaceComposerClient.h>
#include <gui/SurfaceComposerClient.h>

#include <system/window.h>
#include <utils/RefBase.h>

using namespace android;

void fillRGBA8Buffer(uint8_t* img, int width, int height, int stride, int r, int g, int b) {
    for (int y = 0; y < height; y++) {
        for (int x = 0; x < width; x++) {
            uint8_t* pixel = img + (4 * (y*stride + x));
            pixel[0] = r;
            pixel[1] = g;
            pixel[2] = b;
            pixel[3] = 255;
        }
    }
}

int SurfaceTest() {
    status_t err = NO_ERROR;
    int countFrame = 0;
    ANativeWindow_Buffer nativeBuffer;

    sp<ProcessState> proc(ProcessState::self());
    ProcessState::self()->startThreadPool();

    sp<SurfaceComposerClient> surfaceComposerClient = new SurfaceComposerClient();
    err = surfaceComposerClient->initCheck();
    if (err != NO_ERROR) {
        ALOGD("SurfaceComposerClient::initCheck error: %#x\n", err);
        return -1;
    }

#if 0
    // Get main display parameters.
    sp<IBinder> displayToken = SurfaceComposerClient::getInternalDisplayToken();
    if (displayToken == nullptr)
        return -1;

    ui::DisplayMode displayMode;
    const status_t error =
        SurfaceComposerClient::getActiveDisplayMode(displayToken, &displayMode);
    if (error != NO_ERROR)
        return -1;
#endif

    ui::Size resolution(600,800);
    // create the surface
    sp<SurfaceControl> surfaceControl = surfaceComposerClient->createSurface(String8("SurfaceTestDemo2"), resolution.getWidth(),
            resolution.getHeight(), PIXEL_FORMAT_RGBA_8888,
            ISurfaceComposerClient::eFXSurfaceBufferState,
            /*parent*/ nullptr);

    SurfaceComposerClient::Transaction{}
        .setLayer(surfaceControl, std::numeric_limits<int32_t>::max())
#if 0
        .setPosition(surfaceControl, (displayMode.resolution.getWidth() - 200) / 2,
            (displayMode.resolution.getHeight() - 200) / 2)
#else
        .setPosition(surfaceControl, (resolution.getWidth() - 200) / 2,
            (resolution.getHeight() - 200) / 2)
#endif
        .show(surfaceControl)
        .apply();

    int mWidth = resolution.getWidth();
    int mHeight = resolution.getHeight();
    sp<Surface> surface = surfaceControl->getSurface();

    while(1) {
        countFrame = (countFrame+1)%3;
        surface->lock(&nativeBuffer, NULL);
#if 1
        fillRGBA8Buffer((uint8_t *)nativeBuffer.bits, mWidth, mHeight, nativeBuffer.stride,
            countFrame == 0 ? 255 : 0,
            countFrame == 1 ? 255 : 0,
            countFrame == 2 ? 255 : 0);
#else
#endif
        surface->unlockAndPost();
        sleep(1);
    }

    return err;
}

int main() {
    SurfaceTest();
    return 0;
}
```

Android.bp
```shell
cc_defaults {
    name: "demo_defaults",

    cflags: [
        "-Wall",
        "-Werror",
        "-Wunused",
        "-Wunreachable-code",
    ],

    shared_libs: [
        "libbase",
        "libbinder",
        "libcutils",
        "liblog",
        "libutils",
        "libui",
        "libgui",
        "libEGL",
        "libGLESv1_CM",
    ],
}

cc_binary {
    name: "DisplayDemo",
    defaults: ["demo_defaults"],
    srcs: [
        "main.cpp",
    ],

    header_libs: [
	"libgui_headers",
    ],

    cflags: [
        "-Wno-deprecated-declarations",
        "-Wno-unused-parameter"
    ],
}

cc_binary {
    name: "DisplayDemo2",
    defaults: ["demo_defaults"],
    srcs: [
        "main2.cpp",
    ],

    header_libs: [
	"libgui_headers",
    ],

    cflags: [
        "-Wno-deprecated-declarations",
        "-Wno-unused-parameter"
    ],
}
```