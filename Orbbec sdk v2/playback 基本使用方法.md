```c++
#include <iostream>
#include <memory>
#include <filesystem>
#include <opencv2/opencv.hpp>

#include "libobsensor/ObSensor.hpp"

namespace fs = std::filesystem;

int main() {

    std::string filePath = "/home/michall/AAAProjects/RGBD/projects/testproject/data/IR01.bag";

    // 1. create playback device
    std::shared_ptr<ob::PlaybackDevice> playback =
        std::make_shared<ob::PlaybackDevice>(filePath);

    // 2. create pipeline
    std::shared_ptr<ob::Pipeline> pipe =
        std::make_shared<ob::Pipeline>(playback);

    // 3. config
    std::shared_ptr<ob::Config> config =
        std::make_shared<ob::Config>();

    // 4. stop flag
    //播放完成自动暂停，因为有[&],会指向相同地址
    bool playbackStop = false;

    playback->setPlaybackStatusChangeCallback(
        [&](OBPlaybackStatus status) {
            if(status == OB_PLAYBACK_STOPPED) {
                playbackStop = true;
            }
        }
    );

    // 5. enable all streams
    auto sensorList = playback->getSensorList();

    for(uint32_t i = 0; i < sensorList->getCount(); i++) {
        auto sensorType = sensorList->getSensorType(i);
        config->enableStream(sensorType);
    }

    // 6. start pipeline
    pipe->start(config);

    // 7. create output folders
    fs::create_directories("./pictures/left");
    fs::create_directories("./pictures/right");

    int frameIndex = 0;
    int saveIndex = 0;

    while(!playbackStop) {

        auto frameSet = pipe->waitForFrames(1000);
        if(frameSet == nullptr) continue;

        frameIndex++;

        // ✔ 每 3 帧取一次
        if(frameIndex % 3 != 0) continue;

        // 获取 IR 左右帧（关键）
        auto leftFrame  = frameSet->getFrame(OB_FRAME_IR_LEFT);
        auto rightFrame = frameSet->getFrame(OB_FRAME_IR_RIGHT);
        auto leftIr  = leftFrame->as<ob::IRFrame>();
        auto rightIr = rightFrame->as<ob::IRFrame>();

        if(leftFrame && rightFrame) {

            int width  = leftIr->getWidth();   // or rightIr->getWidth()
            int height = leftIr->getHeight();
            cv::Mat leftImg(height, width, CV_8UC1, (void*)leftIr->getData());
            cv::Mat rightImg(height, width, CV_8UC1, (void*)rightIr->getData());

            // file name
            std::string leftPath  = "./pictures/left/"  + std::to_string(saveIndex) + ".png";
            std::string rightPath = "./pictures/right/" + std::to_string(saveIndex) + ".png";

            cv::imwrite(leftPath, leftImg);
            cv::imwrite(rightPath, rightImg);

            std::cout << "Saved frame " << saveIndex << std::endl;

            saveIndex++;
        }
    }

    pipe->stop();

    return 0;
}
```