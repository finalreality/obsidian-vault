```cpp
#include <iostream>
#include <fcntl.h>
#include <sys/ioctl.h>
#include <linux/input.h>
#include <poll.h>
#include <dirent.h>
#include <string>
#include <vector>
#include <map>
#include <unistd.h>
#include <cstring>
#include <mutex>
#include <condition_variable>
#include <atomic>
#include <thread>

#define EVIOCGNAME(len)  _IOR('E', 0x06, char[len])
#define BUFFER_SIZE 1024

using namespace std;

struct TouchEvent {
    int id;
    int x;
    int y;
    bool is_pressed;
};

class InputReader {
public:
    InputReader();
    ~InputReader();
    bool Initialize();
    void ReadEvents();
    void PrintTouchEvent(const TouchEvent& event, bool is_pressed);
    void Stop();

    thread input_thread_;

private:
    bool OpenTouchEventDevice();
    void MonitorInput();
    void ProcessEvents();

    int fd_;
    atomic<bool> running_;
    map<int, TouchEvent> touch_events_;
    vector<struct input_event> event_buffer_;
    struct pollfd pfd_;
    mutex data_mutex_;
    bool touched_;
};

InputReader::InputReader() : fd_(-1), running_(false), touched_(false) {}
InputReader::~InputReader() {
    if (running_) {
        Stop();
    }
    if (fd_ != -1) {
        close(fd_);
    }
}

void InputReader::Stop() {
    running_ = false;
}

bool InputReader::Initialize() {
    if (OpenTouchEventDevice()) {
        input_thread_ = thread(&InputReader::MonitorInput, this);
        running_ = true;
        return true;
    }
    return false;
}

bool InputReader::OpenTouchEventDevice() {
    const string input_dir = "/dev/input/";
    DIR* dir = opendir(input_dir.c_str());
    if (!dir) {
        return false;
    }

    while (true) {
        struct dirent* ent = readdir(dir);
        if (!ent) {
            closedir(dir);
            return false;
        }

        string device_path = input_dir + ent->d_name;
        if (!strstr(ent->d_name, "event")) {
            continue;
        }

        int fd = open(device_path.c_str(), O_RDONLY | O_NONBLOCK);
        if (fd == -1) continue;

        char name[256];
        if (ioctl(fd, EVIOCGNAME(sizeof(name)), name) < 0) {
            close(fd);
            continue;
        }

	std::cout << "Name: " << name << std::endl;

        if (strstr(name, "fts")) {
            fd_ = fd;
            pfd_.fd = fd;
            pfd_.events = POLLIN;
            closedir(dir);
            return true;
        } else {
            close(fd);
        }
    }

    closedir(dir);
    return false;
}

void InputReader::MonitorInput() {
    struct input_event events[BUFFER_SIZE];
    int read_count;

    while (running_.load()) {
        int ret = poll(&pfd_, 1, -1);
        if (ret < 0) {
            continue;
        } else if (ret > 0) {
            while (true) {
                read_count = read(fd_, events, sizeof(struct input_event) * BUFFER_SIZE);

		std::cout << "Read Count: " << read_count << std::endl;

                if (read_count == 0) {
                    break;
                } else if (read_count < 0) {
					std::cout << "Read Count -1" << std::endl;
                    if (errno == EAGAIN || errno == EWOULDBLOCK) {
                        break;
                    } else {
                        continue;
                    }
                }

                lock_guard<mutex> guard(data_mutex_);
                for (size_t i = 0; i < read_count / sizeof(struct input_event); ++i) {
                    event_buffer_.push_back(events[i]);
                }
            }
        }

        ProcessEvents();
    }
}

void InputReader::ProcessEvents() {
    lock_guard<mutex> guard(data_mutex_);
    while (!event_buffer_.empty()) {
        struct input_event event = event_buffer_.front();
        event_buffer_.erase(event_buffer_.begin());

        switch (event.type) {
            case EV_KEY:
		std::cout << "Process Event KEY" << std::endl;
                if (event.code == BTN_TOUCH) {
		std::cout << "Process Event KEY touched_: " << event.value << std::endl;
                    touched_ = (event.value == 1);
                }
                break;

            case EV_ABS:
		std::cout << "Process Event ABS" << std::endl;
                if (touched_) {
                    if (event.code == ABS_X) {
                        int id = 0; // 假设单点触摸，ID为0
                        touch_events_[id].x = event.value;
                        touch_events_[id].is_pressed = touched_;
                    } else if (event.code == ABS_Y) {
                        int id = 0;
                        touch_events_[id].y = event.value;
                        touch_events_[id].is_pressed = touched_;
                    }
                }
                break;

            case EV_SYN:
		std::cout << "Process Event SYN: code: " << event.code << std::endl;
                if (event.code == SYN_REPORT) {
		std::cout << "Process Event 1" << std::endl;
                    if (!touch_events_.empty()) {
		std::cout << "Process Event 2" << std::endl;
                        for (const auto& pair : touch_events_) {
		std::cout << "Process Event 3" << std::endl;
                            const TouchEvent& event = pair.second;
                            if (event.is_pressed) {
                                PrintTouchEvent(event, true);
                            } else {
                                PrintTouchEvent(event, false);
                            }
                        }
                    }
                    touch_events_.clear();
                }
                break;
        }
    }
}

void InputReader::PrintTouchEvent(const TouchEvent& event, bool is_pressed) {
		std::cout << "PrintTouchEvent" << std::endl;
    if (is_pressed) {
        cout << "[Touch Press] ID: 0, X: " << event.x << ", Y: " << event.y << endl;
    } else {
        cout << "[Touch Release] ID: 0" << endl;
    }
}

int main() {
    InputReader reader;
    if (!reader.Initialize()) {
        cerr << "Failed to initialize input reader." << endl;
        return 1;
    }

    cout << "Press Q followed by Enter to exit..." << endl;
    string input;
    while (getline(cin, input)) {
        if (input == "Q" || input == "q") {
            reader.Stop();
            break;
        }
    }

    if (reader.input_thread_.joinable()) {
        reader.input_thread_.join();
    }

    return 0;
}

```