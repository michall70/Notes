SDK Entrypoint (ObSensor.hpp)
Just re-exports sub-headers. In your code, `#include "libobsensor/ObSensor.hpp" gets you the entire SDK.
```c++

Core Classes

1. Context — SDK lifecycle & device enumeration
Context(const char* configPath = "")          // empty = default config
queryDeviceList()                              // → shared_ptr<DeviceList>
enableNetDeviceEnumeration(bool enable)
createNetDevice(ip, port, accessMode)          // → shared_ptr<Device>
setDeviceChangedCallback(callback)
setLoggerSeverity(OBLogSeverity)               // static
setLoggerToFile(severity, directory)           // static
setLoggerToConsole(severity)                   // static
freeIdleMemory()


2. Device — Physical camera representation
getDeviceInfo()                                // → shared_ptr<DeviceInfo>
getSensorList()                                // → shared_ptr<SensorList>
getSensor(OBSensorType)                        // → shared_ptr<Sensor>
setIntProperty(OBPropertyID, int32_t)
getIntProperty(OBPropertyID)                   // → int32_t
setFloatProperty(OBPropertyID, float)
setBoolProperty(OBPropertyID, bool)
isPropertySupported(id, permission)            // → bool
getSupportedPropertyCount()
getSupportedProperty(index)                    // → OBPropertyItem
reboot()
updateFirmware(filePath, callback, async)


3. DeviceInfo — Read device metadata
getName()              getVid()        getPid()
getUid()               getSerialNumber()
getFirmwareVersion()   getHardwareVersion()
getConnectionType()    getIpAddress()
getDeviceType()        getAsicName()


4. DeviceList — List of enumerated devices
getCount()                                   // → uint32_t
getDevice(index)                             // → shared_ptr<Device>
getPid(index)      getVid(index)     getUid(index)
getSerialNumber(index)   getName(index)
getConnectionType(index)  getIpAddress(index)


5. Sensor — Individual sensor on a device
getType()                                    // → OBSensorType
getStreamProfileList()                       // → shared_ptr<StreamProfileList>
start(shared_ptr<StreamProfile>, FrameCallback callback)
stop()
switchProfile(shared_ptr<StreamProfile>)
createRecommendedFilters()                   // → vector<shared_ptr<Filter>>


6. Pipeline — High-level streaming (recommended approach)
Construction:
Pipeline()                                   // auto-selects first device
Pipeline(shared_ptr<Device>)                 // specific device
Pipeline(shared_ptr<PlaybackDevice>)         // from .bag file (what you use)

Lifecycle:
start(shared_ptr<Config>)                    // start stream
start(shared_ptr<Config>, FrameSetCallback)  // start with callback
stop()
waitForFrames(uint32_t timeoutMs=1000)       // → shared_ptr<FrameSet>
waitForFrameset(timeoutMs)                   // alias
getConfig()                                  // → shared_ptr<Config>
getDevice()                                  // → shared_ptr<Device>
enableFrameSync() / disableFrameSync()


7. Config — Pipeline configuration
enableStream(OBStreamType)                   // enable by stream type
enableStream(OBSensorType)                   // enable by sensor type (auto-converts)
enableStream(shared_ptr<StreamProfile>)       // enable with specific profile
enableVideoStream(streamType, width=ANY, height=ANY, fps=ANY, format=ANY)
disableStream(OBStreamType)
disableAllStream()
setAlignMode(OBAlignMode)
setDepthScaleRequire(bool)
setFrameAggregateOutputMode(mode)


8. PlaybackDevice — Play .bag recordings
PlaybackDevice(const string& filePath)       // construct from .bag
pause() / resume()
seek(int64_t timestampMs)
setPlaybackRate(float rate)
getPosition() / getDuration()
setPlaybackStatusChangeCallback(callback)
getPlaybackStatus()
getSensorList()                              // inherited from Device


9. RecordDevice — Record to .bag
RecordDevice(shared_ptr<Device>, file, compressionEnabled=true)
pause() / resume()


10. FrameSet — Container for synchronized frames
getCount()
getFrame(OBFrameType)                        // e.g. OB_FRAME_DEPTH, OB_FRAME_IR
getFrameByIndex(index)
getDepthFrame()                              // → shared_ptr<DepthFrame>
getColorFrame()                              // → shared_ptr<ColorFrame>
getIrFrame()                                 // → shared_ptr<IRFrame>
getPointsFrame()                             // → shared_ptr<PointsFrame>


11. Frame / VideoFrame / subclasses — Frame data
getType()          getFormat()        getIndex()
getData()          getDataSize()
getWidth()         getHeight()        (VideoFrame)
getTimeStampUs()   getSystemTimeStampUs()   getGlobalTimeStampUs()
getStreamProfile()
getMetadata()      getMetadataSize()
hasMetadata(type)  getMetadataValue(type)
is<T>()            as<T>()            // type-safe casting

Frame hierarchy: 
/**
 *  Frame classis inheritance hierarchy:
 *         Frame
 *          |
 *      +-----------+----------+----------+-----------+
 *      |           |          |          |           |
 *   VideoFrame PointsFrame AccelFrame GyroFrame   FrameSet
 *        |
 *   +----+----------+-------------------+
 *   |               |                   |
 * ColorFrame    DepthFrame           IRFrame
 *                                       |
 *                                 +-----+-----+
 *                                 |           |
 *                            IRLeftFrame  IRRightFrame
 */
 
Key frame type constants: OB_FRAME_DEPTH, OB_FRAME_COLOR, OB_FRAME_IR, OB_FRAME_IR_LEFT, OB_FRAME_IR_RIGHT, OB_FRAME_POINTS, OB_FRAME_ACCEL, OB_FRAME_GYRO


12. StreamProfile / StreamProfileList — Stream configuration info
StreamProfile: getType(), getFormat(), getExtrinsicTo(target)
VideoStreamProfile: getFps(), getWidth(), getHeight(), getIntrinsic(), getDistortion()
AccelStreamProfile: getFullScaleRange(), getSampleRate(), getIntrinsic()
GyroStreamProfile: getFullScaleRange(), getSampleRate(), getIntrinsic()
StreamProfileList: getCount(), getProfile(index), getVideoStreamProfile(w, h, format, fps)


13. Filters — Post-processing
Concrete filter classes:
- PointCloudFilter — depth → point cloud
- Align — depth-to-color alignment
- FormatConvertFilter — format conversion
- DecimationFilter — subsample depth
- ThresholdFilter — depth range clipping
- HdrMerge — HDR merge
- SpatialAdvancedFilter / SpatialFastFilter / SpatialModerateFilter — spatial smoothing
- HoleFillingFilter — fill depth holes
- NoiseRemovalFilter — remove scatter noise
- TemporalFilter — temporal smoothing
- EdgeNoiseRemovalFilter / FalsePositiveFilter — advanced cleanup
All filters: process(frame), pushFrame(frame) + setCallBack(cb) async, setConfigValue(name, value), reset(), enable(bool)


Typical Usage Pattern (what main.cpp & extract.cpp do)
1. PlaybackDevice(filePath) — load .bag
2. Pipeline(playback) — bind to pipeline
3. new Config() — create config
4. config->enableStream(sensorType) — for each sensor in playback->getSensorList()
5. pipe->start(config) — start streaming
6. Loop: pipe->waitForFrames(timeout) → get FrameSet
7. frameSet->getFrame(OB_FRAME_IR_LEFT) → specific frame
8. frame->getData(), frame->getWidth(), frame->getHeight() → raw data
9. pipe->stop() — cleanup
```
