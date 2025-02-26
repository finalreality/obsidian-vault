```cpp
#include <iostream>
#include <fstream>
#include <cstdint>
#include <cstring>
#include <linux/input.h>
#include <unistd.h>
#include <fcntl.h>
#include <vector>
#include <string>

// 触摸事件类型
enum class TouchEventType {
    TOUCH_DOWN,
    TOUCH_MOVE,
    TOUCH_UP,
    UNKNOWN
};

// 触摸事件结构
struct TouchEvent {
    TouchEventType type;
    int32_t x;
    int32_t y;
    int32_t pressure; // 可选，表示触摸压力或接触面积

    TouchEvent(TouchEventType t, int32_t xx, int32_t yy, int32_t p = 0)
        : type(t), x(xx), y(yy), pressure(p) {}
};

// 触摸输入类
class TouchInput {
public:
    TouchInput(const std::string& devicePath) {
        device_ = open(devicePath.c_str(), O_RDONLY | O_NONBLOCK);
        if (device_ == -1) {
            throw std::runtime_error("Failed to open device: " + devicePath);
        }
    }

    ~TouchInput() {
        if (device_ != -1) {
            close(device_);
        }
    }

    // 读取并解析下一个触摸事件
    TouchEvent getNextTouchEvent() {
        struct input_event event;
        ssize_t bytesRead = read(device_, &event, sizeof(event));

		if (bytesRead < 0) {
            return TouchEvent(TouchEventType::UNKNOWN, 0, 0);
		}

        if (bytesRead != sizeof(event)) {
            //throw std::runtime_error("Failed to read event: " + std::to_string(bytesRead));
            std::cout<< "Failed to read event: " + std::to_string(bytesRead) << std::endl;
        }

		std::cout << "Event type: " << event.type << "  ";
		std::cout << "Event code: " << event.code << "  ";
		std::cout << "Event value: " << event.value << std::endl;

        // 根据事件类型和代码处理事件
        TouchEventType eventType = TouchEventType::UNKNOWN;
        int32_t x = 0, y = 0, pressure = 0;

        switch (event.type) {
            case EV_ABS:
                // 处理绝对坐标事件
                switch (event.code) {
                    case ABS_MT_POSITION_X:
					std::cout << "ABS_MT_POSITION_X: " << event.value << std::endl;
                        x = event.value;
                        break;
                    case ABS_MT_POSITION_Y:
					std::cout << "ABS_MT_POSITION_Y: " << event.value << std::endl;
                        y = event.value;
                        break;
                    case ABS_MT_PRESSURE:
                        pressure = event.value;
                        break;
                    case ABS_MT_TRACKING_ID:
                        // 跟踪ID用于区分多个触摸点
                        trackingId_ = event.value;
                        break;
                    default:
                        break;
                }
                break;

            case EV_SYN:
                // 同步事件表示一组事件的完成
                if (event.code == SYN_REPORT) {
                    // 根据之前的ABS_MT_TRACKING_ID和坐标值生成触摸事件
					std::cout << "trackingId_: " << trackingId_ << std::endl;
                    if (trackingId_ >= 0) {
                        if (x != lastX_ || y != lastY_) {
                            eventType = TouchEventType::TOUCH_MOVE;
                        } else if (pressure > 0) {
                            eventType = TouchEventType::TOUCH_DOWN;
                        } else {
                            eventType = TouchEventType::TOUCH_UP;
                        }
                        lastX_ = x;
                        lastY_ = y;
		std::cout << "111 lastX: " << x << " lastY: " << y << std::endl;
                    } else if(trackingId_ == -1) {
                        if (pressure > 0) {
                            eventType = TouchEventType::TOUCH_DOWN;
                        } else {
                            eventType = TouchEventType::TOUCH_UP;
                        }
		std::cout << "222 lastX: " << x << " lastY: " << y << std::endl;
					}
                }
                break;

            default:
                break;
        }

        // 返回触摸事件（仅在有效事件时）
        if (eventType != TouchEventType::UNKNOWN) {
            return TouchEvent(eventType, lastX_, lastY_, pressure);
        } else {
            // 如果没有有效事件，则返回一个默认构造的TouchEvent（可能不是很有用）
            return TouchEvent(TouchEventType::UNKNOWN, 0, 0);
        }
    }

private:
    int device_;
    int32_t trackingId_ = -1; // 当前跟踪的触摸ID
    int32_t lastX_ = 0;       // 上一个X坐标
    int32_t lastY_ = 0;       // 上一个Y坐标
};

int main() {
    std::string devicePath = "/dev/input/event6"; // 替换X为实际的设备编号

    try {
        TouchInput touchInput(devicePath);

        while (true) {
            TouchEvent touchEvent = touchInput.getNextTouchEvent();

            // 输出触摸事件信息
            switch (touchEvent.type) {
                case TouchEventType::TOUCH_DOWN:
                    std::cout << "Touch Down at (" << touchEvent.x << ", " << touchEvent.y << ") with pressure " << touchEvent.pressure << std::endl;
                    break;
                case TouchEventType::TOUCH_MOVE:
                    std::cout << "Touch Move to (" << touchEvent.x << ", " << touchEvent.y << ") with pressure " << touchEvent.pressure << std::endl;
                    break;
                case TouchEventType::TOUCH_UP:
                    std::cout << "Touch Up at (" << touchEvent.x << ", " << touchEvent.y << ")" << std::endl;
                    break;
                default:
                    break;
            }

            usleep(10000); // 每10毫秒读取一次事件以减少CPU使用率
        }
    } catch (const std::exception& e) {
	    std::cout << "Exception: " << e.what() << std::endl;
    }
}

```