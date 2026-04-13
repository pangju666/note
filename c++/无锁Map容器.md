```cpp
#include <iostream>
#include <unordered_map>
#include <atomic>
#include <mutex>
template <typename KeyType, typename ValueType>
class ThreadSafeMap {
public:
ThreadSafeMap() {}
void insert(const KeyType& key, const ValueType& value) {
auto new_entry = std::make_shared<std::pair<KeyType, ValueType>>(key, value);
while (true) {
auto snapshot = map_.load();
auto itr = snapshot.find(key);
if (itr != snapshot.end()) {
break;
}
auto new_map = snapshot;
new_map.insert(std::make_pair(key, new_entry));
if (map_.compare_exchange_strong(snapshot, new_map)) {
break;
}
}
}
bool find(const KeyType& key, ValueType& value) {
auto snapshot = map_.load();
auto itr = snapshot.find(key);
if (itr == snapshot.end()) {
return false;
}
value = *(itr->second);
return true;
}
bool remove(const KeyType& key) {
auto snapshot = map_.load();
auto itr = snapshot.find(key);
if (itr == snapshot.end()) {
return false;
}
auto new_map = snapshot;
new_map.erase(itr);
return map_.compare_exchange_strong(snapshot, new_map);
}
private:
std::atomic<unordered_map<key, value>*> map_;
}
```