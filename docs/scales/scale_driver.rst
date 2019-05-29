.. _scale_driver-index:

电子秤驱动
============================



搭建QT交叉编译环境
----------------------------

	1. 将fsl-imx-fb-glibc-x86_64-meta-toolchain-qte-cortexa9hf-neon-toolchain-qte-4.1.15-2.1.0.sh拷贝到Ubuntu；
	
	2. 进行该文件所在目录，执行命令：sudo ./fsl-imx-fb-glibc-x86_64-meta-toolchain-qte-cortexa9hf-neon-toolchain-qte-4.1.15-2.1.0.sh；
	
	3. 进入/opt/fsl-imx-fb/4.1.15-2.1.0/；
	
	4. 执行：source environment-setup-cortexa9hf-neon-poky-linux-gnueabi；
	
	5. 进入QT工程目录，执行：1) qmake  xxxx.pro    2) make；


电子秤驱动编译使用
----------------------------

	1.如果只需要在串口终端里观察重量数据，则需把GetWeightData函数里的counter自减语句注释掉；
	
	2.进入hx711工程目录，执行：1）qmake hx711.pro    2）make；

	3.通过把编译生成的hx711可执行文件拷贝到upan里，待板子启动起来之后，挂载到blurr板上；

	4.执行：1）mount /dev/你的U盘符 /mnt（盘符通过命令fdisk -l查看）
	             2）cd /mnt     3）./hx711
	
	
传感器与blurr板接线
----------------------------

	1.传感器绿线接hx711模块的A+，白线接A-，红线接E+，黑线接E-，屏蔽线不用管；
	
	2.hx711模块另一侧的线插在blurr板8位开关下面的uart3插槽上(即靠近网线插口一侧的那个插槽)；


