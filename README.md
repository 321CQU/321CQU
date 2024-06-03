![@321CQU](https://avatars.githubusercontent.com/u/143437390?s=200&v=4)

- [什么是321CQU](#什么是321CQU)
  - [体验一下](#体验一下)
- [技术架构](#技术架构)
- [开发文档](#开发文档)
- [开源协议](#开源协议)

## 什么是321CQU

321CQU是重庆大学最大的校园服务平台，提供适用于微信小程序、iPhone和智能手表等多端的客户端服务。致力于给每一个CQUer提供便捷的信息查询服务。目前已经覆盖了成绩查询、课表查询、志愿时长查询、书籍借阅情况查询、考试安排查询、往年课程成绩分布查询、空教室查询、宿舍水电费及校园卡余额查询和校园论坛等多项服务。

总所周知，将大象放进冰箱需要三步，我们的愿景是：重大师生不需要在多个页面间寻找信息，像默数“321”一样迅速便捷的获取想知道的信息。

项目可读性、拓展性优秀，并提供CI\CD服务。采用微服务架构，使用docker容器化各微服务，提供统一网关接口，网关通过grpc提供具体业务逻辑，保障项目的灵活性，同时兼顾多语言开发。

### 体验一下

打开微信，搜索“321CQU”

## 技术架构



![image-20240603160156130](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20240603160156130.png)

其中，各个微服务之间以及API_Gateway与微服务之间以grpc连接，API_Gateway对API提供统一入口。

## 开发文档

### Public_Repository

> [321CQU/Public_Repository](https://github.com/321CQU/Public_Repository)

​	321CQU后端公共仓库，在部署时将通过 volumes 挂载到每个容器内，为各个微服务提供共享资源。请注意，这是本项目的重要仓库，包含以下内容：

- 后端 gRPC 所需的全部 protobuf 文件
- 部分被多个微服务依赖的工具类（例如，抽象数据库管理类、配置文件读取类、单例模式元类）
- 定义环境变量

### API_Gateway

> [321CQU/API_Gateway](https://github.com/321CQU/API_Gateway)

​	使用 Sanic 框架构建了一个 API 网关服务，对API请求提供统一入口。并提供了异常处理、权限校验、请求\返回参数校验等功能。

#### authorization.py：

​	用于实现登录和鉴权相关，具体功能：

- 定义`LoginApplyType`类，表示不同的API请求类型，并根据不同请求类型提供不同的检查API key的方法。
- 使用 Pydantic 定义了 Token Payload、登录请求/响应、刷新 Token 请求/响应的模型。
- 提供登录和刷新 Token 的 API。
- 实现 API 权限校验装饰器：`authorized` 装饰器，用于检查请求的权限和 Token 有效性。

#### campus_life.py

​	用于处理与校园生活相关的 API 请求，具体功能：

- 获取校园卡信息。`/campus_lift/card`
- 获取最近30天的账单信息。`/campus_lift/bill/card`
- 获取宿舍水电费信息。`/campus_lift/bill/dorm_energy`

> 其他blueprint不再列举，请自行查阅[api文档](https://api.321cqu.com/docs)。

### Control_Center：

​	使用 Swift 编写的微服务，利用 Swift 的并发特性和 NIO 框架来处理异步任务。主要提供了主页信息的缓存和获取功能。它通过 gRPC 接口对外提供服务，功能如下：

1. 主页信息的缓存管理：将从数据库中获取的主页信息（轮播图信息）缓存到内存中，避免频繁读数据库。同时定期刷新以保证数据同步。
2. 获取主页（轮播图）信息，`getHomepageInfos`:返回缓存中的主页信息，如果缓存未初始化，则从数据库中获取主页信息并初始化缓存。
3. 强制刷新主页（轮播图）信息缓存，`forceRefreshHomepageInfoCache`：从数据库中获取最新的主页信息并更新缓存，将返回最新的主页信息。

### Mycqu_Service

> [321CQU/Mycqu_Service](https://github.com/321CQU/Mycqu_Service)
>
> 基于pymycqu基础上提供微服务，采用的pymycqu为进行了部分修改的版本，详见[pymycqu](https://github.com/ZhuLegend/pymycqu/tree/_321CQU_custom)的`_321CQU_custom`分支。

​	使用 gRPC 和 asyncio 编写的 Python 微服务，定义并启动了一个 gRPC 服务器，提供 Mycqu、Card 和 Library 三个服务。需要注意的是，本微服务并不是操作的具体执行者，具体执行逻辑位于[pymycqu](https://github.com/ZhuLegend/pymycqu/tree/_321CQU_custom)库，本微服务仅增加了通过缓存机制来管理和复用客户端实例的功能，以减少不必要的登录操作。

- #### Card服务：

  用于处理与校园卡相关的请求，提供以下接口：

  - `FetchCard`：获取用户校园卡信息并返回。
  - `FetchBills`：获取用户校园卡账单信息并返回。
  - `FetchEnergyFee`：获取用户的水电费信息并返回。

- #### Library服务：

  用于处理与图书馆相关的请求，提供以下接口：

  - `FetchBorrowBook`：请求参数获取当前或历史借阅的图书信息。
  - `RenewBook`:进行图书续借操作。

- #### Mycqu服务：

  用于处理与Mycqu相关的多种请求，提供以下接口：

  - `FetchUser`:获取用户基本信息(姓名、学号、身份等校园信息)。
  - `FetchEnrollCourseInfo`:获取用户选课信息。
  - `FetchEnrollCourseItem`:获取单个的选课项信息。
  - `FetchExam`：获取用户考试信息。
  - `FetchAllSession`:获取所有的学期信息（包括学期ID，学期年份，是否为秋季学期等）。
  - `FetchCurrSessionInfo`:获取当前学期信息具体信息。
  - `FetchAllSessionInfo`:获得所有学期的详细信息。
  - `FetchCourseTimetable`：获得课程表信息。
  - `FetchEnrollTimetable`：获得选课信息。
  - `FetchScore`：获得具体成绩信息。
  - `FetchGpaRanking`：获得Gpa以及排名信息。

  > 具体使用方法请查阅[mycqu_service.proto](https://github.com/321CQU/Public_Repository/blob/main/micro_services_protobuf/mycqu_service/mycqu_service.proto)信息。

### Edu_Admin_Center

> [321CQU/Edu_Admin_Center](https://github.com/321CQU/Edu_Admin_Center)

​	使用 gRPC 和 asyncio 编写的 Python 微服务，定义并启动了一个 gRPC 服务器。负责对Mycqu_Service提供进一步封装。进一步增加了以下功能：

- 学期信息缓存机制。因为所有人的学期信息都是一致的，所以缓存学期信息是很有必要的。

- 提供更便携易用且高效的课表查询接口，只需要传入offset即可获取指定学期的课表信息。自动调用Mycqu_Service的rpc请求获取学期列表和当前学期信息，通过偏移量计算目标学期信息，调用指定接口获取课表信息，并格式化后返回。
- 将成绩信息写入数据库。可以对成绩分布进行统计，展示课程成绩分布等信息。



### 开源协议

本项目遵循AGPL-3.0 license开源协议，我们欢迎任何形式的贡献！