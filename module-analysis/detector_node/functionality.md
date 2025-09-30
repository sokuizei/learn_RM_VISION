# Armor_detector里的detector_node功能  

## 1.主要函数构成（从构造函数来找）  
### 1.1 构造函数流程  
```cpp
//构造函数
ArmorDetectorNode::ArmorDetectorNode(const rclcpp::NodeOptions & options)
: Node("armor_detector", options){}
```
首先这个构造函数使用了`rclcpp::NodeOptions & options`，作用后续补充....
然后初始化基类Node，名字为`"armor_detector"`
构造函数执行了以下函数：  
1.
```cpp
 detector_ = initDetector();
 ```
  `detector->classifier =`
 这里初始化函数返回一个**std::unique_ptr<Detector>的指针detector**，函数**初始化了**Detertor构造函数所需参数`（binary_thres, detect_color, l_params, a_params）`
 和Detector成员函数std::unique_ptr<NumberClassifier> classifier所需参数`（model_path, label_path, threshold, ignore_classes）`；  
 2.
```cpp
armors_pub_ = this->create_publisher<auto_aim_interfaces::msg::Armors>(
    "/detector/armors", rclcpp::SensorDataQoS());
```
创建一个发布器，发布消息类型为`auto_aim_interfaces::msg::Armors`，发布主题为`"/detector/armors"`  *PS:这里Armors是自定义的消息类型，在`auto_aim_interfaces/msg/Armors.msg`中定义，类似的后续不再陈述。*  
3.
```cpp
marker_pub_ =
    this->create_publisher<visualization_msgs::msg::MarkerArray>("/detector/marker", 10);
```  
创建一个发布器，发布消息类型为`visualization_msgs::msg::MarkerArray`，发布主题为`"/detector/marker"`，这里我还不太懂ros2 里的marker是什么，搞懂后补充于后文。  
4.  
debug这里声明了参数判断是否进行调试，还设置了个参数事件处理，用于根据参数变化来开启和关闭响应的话题发布  
*这里的话题在`void ArmorDetectorNode::createDebugPublishers()`中介绍*  
5.
```cpp
cam_info_sub_ = this->create_subscription<sensor_msgs::msg::CameraInfo>(
    "/camera_info", rclcpp::SensorDataQoS(),
    [this](sensor_msgs::msg::CameraInfo::ConstSharedPtr camera_info) {
      cam_center_ = cv::Point2f(camera_info->k[2], camera_info->k[5]);
      cam_info_ = std::make_shared<sensor_msgs::msg::CameraInfo>(*camera_info);
      pnp_solver_ = std::make_unique<PnPSolver>(camera_info->k, camera_info->d);
      cam_info_sub_.reset();
    });
```
这里创建了一个订阅器，订阅话题为`"/camera_info"`，消息类型为`sensor_msgs::msg::CameraInfo`，回调函数为`[this](sensor_msgs::msg::CameraInfo::ConstSharedPtr camera_info)`，回调函数的作用是获取相机内参，并初始化PnPSolver，然后关闭订阅器。*这里订阅完就关闭是因为相机内参信息只需获取一次即可*  
6.
```cpp
 img_sub_ = this->create_subscription<sensor_msgs::msg::Image>(
    "/image_raw", rclcpp::SensorDataQoS(),
    std::bind(&ArmorDetectorNode::imageCallback, this, std::placeholders::_1));
```
创建一个订阅器，订阅话题为`"/image_raw"`，消息类型为`sensor_msgs::msg::Image`，回调函数的作用是处理图像数据。  
### 1.2 另一个重要函数imagecallback及其附属




