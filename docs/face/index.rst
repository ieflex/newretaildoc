.. _face:

人脸识别
============================


V4L2
----------------------------

(I) **介绍**

V4L2是Video for Linux API version 2的简称，是linux中关于视频设备的内核驱动。在Linux中，视频设备是设备文件，可以像访问普通文件一样对其进行读写。


(#) **支持的接口**

可以支持多种设备,它可以有以下几种接口:
1. 视频采集接口(video capture interface)
	这种应用的设备可以是高频头或者摄像头.V4L2的最初设计就是应用于这种功能的。

#. 视频输出接口(video output interface)
	可以驱动计算机的外围视频图像设备——像可以输出电视信号格式的设备。

#. 直接传输视频接口(video overlay interface)
	它的主要工作是把从视频采集设备采集过来的信号直接输出到输出设备之上，而不用经过系统的CPU。

#. 视频间隔消隐信号接口(VBI interface)
	它可以使应用可以访问传输消隐期的视频信号。

#. 收音机接口(radio interface)
	可用来处理从AM或FM高频头设备接收来的音频流。


(#) **采集方式**

1. 打开视频设备
::
	int fd = open("/dev/video0", O_RDWR);

#. 设定属性。视频设备打开后，通常使用 ioctl 函数获取和设置视频设备的属性。
::
	struct v4l2_capability cap;
	ioctl(fd, VIDIOC_QUERYCAP, &cap)

#. 设定采集方式。根据需要设置视频设备的采集方式为V4L2_MEMORY_USERPTR或V4L2_MEMORY_MMAP等。
::
	struct v4l2_requestbuffers req;
	memset(&req, 0, sizeof(req));
	req.count	= 4;
	req.type	= V4L2_BUF_TYPE_VIDEO_CAPTURE;
	req.memory	= V4L2_MEMORY_USERPTR;
	ioctl(fd, VIDIOC_REQBUFS, &req);

#. 启动数据采集
::
	enum v4l2_buf_type type = V4L2_BUF_TYPE_VIDEO_CAPTURE;
	ioctl(fd, VIDIOC_STREAMON, &type);

#. 循环采集数据并处理
::
	fd_set fds;
	struct timeval tv;
	FD_ZERO(&fds);
	FD_SET(fd, &fds);
	tv.tv_sec	 = 2;
	tv.tv_usec = 0;
	if(select(fd + 1, &fds, NULL, NULL, &tv) > 0)
	{
		struct v4l2_buffer buf;
		memset(&buf, 0, sizeof(buf));
		buf.type = V4L2_BUF_TYPE_VIDEO_CAPTURE;
		buf.memory = V4L2_MEMORY_USERPTR;
		if(0 == ioctl(fd, VIDIOC_DQBUF, &buf))
		{
			// 处理图像数据
		}
		ioctl(fd, VIDIOC_QBUF, &buf);
	}

#. 停止数据采集
::
	enum v4l2_buf_type type = V4L2_BUF_TYPE_VIDEO_CAPTURE;
	ioctl(fd, VIDIOC_STREAMOFF, &type);

#. 关闭视频设备
::
	close(fd);


(#) **示例及文档**

:示例: https://linuxtv.org/downloads/legacy/video4linux/API/V4L2_API/v4l2spec/capture.c
:文档: https://linuxtv.org/downloads/v4l-dvb-apis/uapi/v4l/v4l2.html


opencv
----------------------------

(I) **介绍**

OpenCV（Open Source Computer Vision Library）是一个开源的计算机视觉和机器学习软件库。OpenCV的建立是为了加速计算机视觉在商业产品中的应用。OpenCV采用BSD开源协议，所以对非商业应用和商业应用都是免费（FREE）的。
OpenCV提供了C++、Python、Java和Matlab等接口，支持Windows、Linux、Android和Mac操作系统。OpenCV主要倾向于实时视觉应用程序，并在可用时利用MMX和SSE指令以提高运算速度。
OpenCV包含有2500多个优化算法，其中包括一系列经典的和最先进的计算机视觉和机器学习算法，这些算法可用于检测和识别人脸、识别对象、对视频中的人类行为进行分类、跟踪摄像机运动、跟踪运动对象、提取对象的3D模型、从立体摄像机中生成3D点云、将图像拼接在一起以生成整个场景的高分辨率图像、从图像数据库中查找相似图像、从图像中去除红眼、跟踪眼睛运动、识别场景并建立标记以覆盖场景等。OpenCV拥有超过47000人的用户群和超过1800万的下载量，广泛用于公司、研究团体和政府机构。


(#) **环境搭建（使用qt作为开发环境）**

在 https://github.com/opencv/opencv 下载opencv.zip
在 https://github.com/opencv/opencv_contrib 下载opencv_contrib.zip
人脸识别需要用到opencv_contrib.zip，如果只是进行人脸检测，不需要安装。
::
	1）在虚拟机中安装 Ubuntu 18.04.2，磁盘大小建议至少40G（编译 opencv 生成的文件会占用很大的空间）。
	2）对 Ubuntu 虚拟机进行设置，“硬件”->“USB控制器”->“USB兼容性” 设置为 “USB 3.0”，否则可能无法读取摄像头的图像数据。
	3）安装 OpenCV 依赖的库
		3.1）安装 cmake
			sudo apt-get install cmake
		3.2）系统自带 build-essential
			sudo apt-get install build-essential
		3.3）安装 g++
			sudo apt install g++
		3.3）程序中使用的 OpenCV 用不到下面的功能，不安装也可以
			sudo apt-get install libgtk2.0-dev libavcodec-dev libavformat-dev libjpeg.dev libtiff4.dev libswscale-dev libjasper-dev
	4）将 opencv.zip 和 opencv_contrib.zip 拷贝到 Ubuntu 系统并解压。然后执行如下命令：
		cd opencv_contrib
		git checkout 4.0.0-rc
		cd ../opencv
		git checkout 4.0.0-rc
		mkdir build
		cd build
		cmake -D CMAKE_BUILD_TYPE=RELEASE -D CMAKE_INSTALL_PREFIX=/home/software/opencv -D OPENCV_EXTRA_MODULES_PATH=../../opencv_contrib/modules/ ..
		make
		sudo make install
	大约需要2个小时的时间（给虚拟机增加内存，编译速度应该会快一些儿），完成之后，OpenCV 会被安装到 /home/software/opencv 目录下。
	5）把opencv的so库加入到环境变量：
		5.1）创建文件：sudo gedit /etc/ld.so.conf.d/opencv.conf
		5.2）输入如下内容并保存：
			/home/software/opencv/lib
		5.3）sudo ldconfig
	6）安装 qtCreator。
		sudo apt-get install qt5-default qtcreator
	7）备注：
		7.1）需要在 qtCreator 的工程文件（.pro）中添加如下代码（第2行 INCLUDEPATH 的值需要根据安装的 OpenCV 的版本进行修改）：
			CONFIG += C++11
			INCLUDEPATH += /home/software/opencv/include/opencv4/
			LIBS += /home/software/opencv/lib/libopencv_*.so


(#) **人脸检测**

使用“摄像头 + OpenCV”实现人脸检测的基本步骤为：
::
	1）打开摄像头。
		cv::VideoCapture cap(0);
	2）加载 OpenCV 自带的人脸检测分类器 haarcascade_frontalface_alt.xml。
		cv::CascadeClassifier face_cascade;
		face_cascade.load("haarcascade_frontalface_alt.xml");
	3）从摄像头读取图像。
		cv::Mat frame;
		cap.read(frame);
	4）对获取的图像进行预处理操作，主要是使用 cvtColor 函数对图像进行灰度化处理。
		cv::Mat frame_gray;
		cv::cvtColor(frame, frame_gray, cv::COLOR_BGR2GRAY);
	5）使用 detectMultiScale 函数进行人脸检测。
		std::vector<cv::Rect> faces;
		face_cascade.detectMultiScale(frame_gray, faces, 1.2, 3, 0, cv::Size(120, 120), cv::Size(300, 300));
	6）获取检测到的人脸个数。
		size_t face_num = faces.size();
	7）由 OpenCV 从摄像头获取的图像创建 QImage。
		cv::Mat frame_rgb;
		cv::cvtColor(frame, frame_rgb, cv::COLOR_BGR2RGB);
		QImage img = QImage((const unsigned char*)frame_rgb.data, frame_rgb.cols, frame_rgb.rows, frame_rgb.cols * frame_rgb.channels(), QImage::Format_RGB888);
	6）获取检测到的人脸个数。
		size_t face_num = faces.size();
	7）如果检测到人脸，使用 rectangle 函数在图像上绘制矩形框。
		for (size_t idx = 0; idx < face_num; idx++)
		{
			cv::rectangle(frame_rgb, faces[idx], cv::Scalar(30, 255, 30), 2, 8, 0);
		}
	8）获取检测到的人脸图像。
		cv::Mat mat_face = frame(faces[0]);
	9）使用 imwrite 函数保存人脸图像到文件。
		cv::imwrite("./整张图像.jpg", frame);
		cv::imwrite("./人脸图像.jpg", mat_face);

