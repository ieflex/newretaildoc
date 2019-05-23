.. _skids-index:

小程序开锁
============================

	小程序开锁采用 Python 语言实现，主要涉及到 **MQTT协议**、**Wifi连接**等。

MQTT协议
----------------------------

	MQTT(消息队列遥测传输)是ISO 标准(ISO/IEC PRF 20922)下基于发布/订阅范式的消息协议。它工作在TCP/IP协议族上，是为硬件性能低下的远程设备以及网络状况糟糕的情况下而设计的发布/订阅型消息协议。
	
	MQTT协议是为大量计算能力有限，且工作在低带宽、不可靠的网络的远程传感器和控制设备通讯而设计的协议，它具有以下主要的几项特性：
	
		1、使用发布/订阅消息模式，提供一对多的消息发布，解除应用程序耦合；
		
		2、对负载内容屏蔽的消息传输；
		
		3、使用 TCP/IP 提供网络连接；
		
		4、有三种消息发布服务质量：
		
			+ “至多一次”，消息发布完全依赖底层 TCP/IP 网络。会发生消息丢失或重复。这一级别可用于如下情况，环境传感器数据，丢失一次读记录无所谓，因为不久后还会有第二次发送。
			
			+ “至少一次”，确保消息到达，但消息重复可能会发生。
			
			+ “只有一次”，确保消息到达一次。这一级别可用于如下情况，在计费系统中，消息重复或丢失会导致不正确的结果。
		
		5、小型传输，开销很小（固定长度的头部是 2 字节），协议交换最小化，以降低网络流量；
		
		6、使用 Last Will 和 Testament 特性通知有关各方客户端异常中断的机制。


WiFi连接
----------------------------

	在 Skids 开发板上连接 WiFi 的代码如下：
	::
		def connectWifi(ssid, passwd):
			global wlan
			
			wlan = network.WLAN(network.STA_IF)			#create a wlan object
			wlan.active(True)							#Activate the network interface
			wlan.disconnect()							#Disconnect the last connected WiFi
			wlan.connect(ssid, passwd)					#connect wifi
			while(wlan.ifconfig()[0] == '0.0.0.0'):
				time.sleep(1)


主要代码
----------------------------

	1. 导入需要的库
	::
		import machine
		from umqtt.simple import MQTTClient
		import network
		import json
		import time

	2. 定义变量
	::
		SSID			= ""										#WiFi 名称
		PASSWORD		= ""										#WiFi 密码
		
		SHOPID			= ""										#shop id
		SERVER			= ""										#MQTT 服务器 ip
		SERVER_PORT		= 3881										#MQTT 服务器端口
		OPEN_DELAY_MS	= 5000										#开门持续时间（时间结束后，自动关门）
		CHECK_DELAY_MS	= 50										#在开门状态下，检查 MQTT 消息的间隔时间
		CLIENT_ID		= "5B6A23C4-183B-48C9-BD6D-154AAC80D5C3"	#MQTT 客户端 id
		username		= None										#MQTT 用户名
		password		= None										#MQTT 密码
		
		OUTPUT_OPEN		= 0			#开门为低电平
		OUTPUT_CLOSE	= 1			#关门为高电平

		TOPIC			= "TOPIC"
		mqttClient		= None
		flag_opening	= False
		close_time		= None
		flag_new_msg	= False
		flag_quit		= False
		wlan			= None

	3. 初始化门的状态为关闭
	::
		OUTPUT = machine.Pin(21, machine.Pin.OUT)		#初始化 Pin
		OUTPUT.value(OUTPUT_CLOSE)						#程序启动后先关门

	4. 开门函数
	::
		def open_door():								#开门时调用
		global close_time
		global OPEN_DELAY_MS
		global flag_opening
		global OUTPUT
		global OUTPUT_OPEN

		if flag_opening:
			close_time = time.ticks_ms() + OPEN_DELAY_MS
			print("door is already opened, update close time", time.ticks_ms())
		else:
			close_time = time.ticks_ms() + OPEN_DELAY_MS
			flag_opening = True
			OUTPUT.value(OUTPUT_OPEN)
			print("open_door", time.ticks_ms())

	5. 关门函数
	::
		def close_door():								#关门时调用
		global flag_opening
		global OUTPUT
		global OUTPUT_OPEN

		if flag_opening:
			flag_opening = False
			OUTPUT.value(OUTPUT_CLOSE)
			print("close_door", time.ticks_ms())
		else:
			print("door is already closed", time.ticks_ms())

	6. MQTT消息回调函数
	::
		def msg_callback(topic, msg):
		global flag_new_msg
		global TOPIC
		global SHOPID
		
		flag_new_msg = True
		
		if topic != TOPIC:
			return
		
		msg_json = None;
		try:
			msg_json = json.loads(msg)
		except:
			msg_json = None;
		
		if None == msg_json:
			return
		
		if msg_json.get("id") != SHOPID:
			return
		
		action = msg_json.get("action")
		if "open" == action:
			open_door()
		elif "close" == action:
			close_door()

	7. 连接 WiFi 函数
	::
		def connectWifi(ssid, passwd):
		global wlan
		
		wlan = network.WLAN(network.STA_IF)			#create a wlan object
		wlan.active(True)							#Activate the network interface
		wlan.disconnect()							#Disconnect the last connected WiFi
		wlan.connect(ssid, passwd)					#connect wifi
		while(wlan.ifconfig()[0] == '0.0.0.0'):
			time.sleep(1)

	8. 主循环
	::
		while not flag_quit:
			try:
				mqttClient	= None
				
				connectWifi(SSID,PASSWORD)
				server = SERVER
				mqttClient = MQTTClient(CLIENT_ID, server, SERVER_PORT, username, password)		#create a mqtt client
				mqttClient.set_callback(msg_callback)											#set callback
				mqttClient.connect()															#connect mqtt
				mqttClient.subscribe(TOPIC)														#client subscribes to a topic
				print("Connected to %s, subscribed to %s topic" % (server, TOPIC))

				while True:
					if flag_opening:
						flag_new_msg = False
						mqttClient.check_msg()
						
						if flag_opening and (time.ticks_ms() >= close_time):
							close_door()
						elif not flag_new_msg:
							time.sleep_ms(CHECK_DELAY_MS)
					else:
						mqttClient.wait_msg()													#wait message
			
			except KeyboardInterrupt:
				flag_quit = True;
			
			except:
				pass
			
			finally:
				if(mqttClient is not None):
					mqttClient.disconnect()
				wlan.disconnect()
				wlan.active(False)
		
		print("exit!")

