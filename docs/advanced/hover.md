# Hover

> 1 Q&A · source: AE plugin dev community Discord

### How do you handle Custom UI mouse-leave events to un-hover buttons?

AE has a known bug where no PF_Event is sent when the mouse leaves the Custom UI area, causing hover states to persist. A workaround is to use a debounce pattern: create a timer callback that triggers after a delay (e.g., 2 seconds). On every mouse-over event, reset the timer. When the timer fires (meaning no mouse-over received recently), trigger an UpdateUI on the main thread to force a Custom UI re-render that resets the hover state.

```cpp
template <typename Func, typename... Args>
std::function<void(Args...)> debounce(Func func, std::chrono::milliseconds delay) {
    std::shared_ptr<std::chrono::steady_clock::time_point> lastCallTime =
        std::make_shared<std::chrono::steady_clock::time_point>();
    return [=](Args... args) {
        *lastCallTime = std::chrono::steady_clock::now();
        std::thread([=]() {
            std::this_thread::sleep_for(delay);
            if (std::chrono::steady_clock::now() - *lastCallTime >= delay) {
                func(args...);
            }
        }).detach();
    };
}
```

*Tags: `custom-ui`, `debounce`, `hover`, `mouse-events`, `workaround`*

---
