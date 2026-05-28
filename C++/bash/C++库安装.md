```bash
cd ~/Downloads
unzip DBoW2-master.zip
cd DBoW2-master
mkdir build && cd build
cmake ..
make -j$(nproc)
sudo make install
```
- 最后`sudo make install`要在`build`文件夹中进行！
---
````bash
make -j$(nproc)
```
调用所有CPU核心，加快并行编译