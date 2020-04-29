.. _snake:

服务端核心代码
============================

基本原理
----------------------------

- **调用流程**

  在odoo.service启动的同时调用新零售服务主线程，服务启动后负责管理新零售整体业务逻辑，初始化电子标签，电子秤等信息。

- **程序基本流程**

  + 启动新零售业务线程
  + 启动UDP监控
  + 初始化业务操作


代码分析
----------------------------

- **启动新零售业务线程**
::

        t1 = StartSocketServer()
        t1.setDaemon(True)
        t1.start()

- **启动UDP监控**
::

    def run(self):
        udp = UdpServer()
        udp.setDaemon(True)
        ip = udp.get_host_ip()
        udp.start()
        # 该线程要执行的任务
        # Create the server, binding to localhost on port 8686
        SocketServer.ThreadingTCPServer.request_queue_size = socket.SOMAXCONN
        SocketServer.ThreadingTCPServer.allow_reuse_address = True
        server = SocketServer.ThreadingTCPServer((ip, PORT), OdooTCPHandler)
        server.daemon = True
        server.allow_reuse_address = True
        # 使用非阻塞方式，需要改动数据接收方式
        # server.socket.setblocking(0)
        # Activate the server; this will keep running until you
        # interrupt the program with Ctrl-C
        server.serve_forever()

- **初始化业务操作**
::

class OdooTCPHandler(SocketServer.BaseRequestHandler):
    """
    处理接受到的硬件Tcp请求
    """

    def handle(self):
        # self.request is the TCP socket connected to the client
        address, pid = self.client_address
        self.mqttdoor = None
        self.mqttCheck = None
        self.health = None
        
        print('%s use %d connected!' % (address, pid))
        _logger.info("%s use %d connected!", address ,pid)
        while True:  # 通讯循环
            try:
                receiveData = odoo.api.receice_data(self.request)
                print receiveData
                # 保障不处理空数据
                if not receiveData:
                    break
                print receiveData['head']['action']
                action = receiveData['head']['action']
                if action == 'login':
                    # 连接数据库并获取
                    self.uniqueId = receiveData['head']['equipId']  # random.randint(1, 20)
                    setLogin(self.uniqueId, address)
                    odoo.api.send_data(self.request, '', self.uniqueId, 'loginReturn')
                    # 保存客户端
                    odoo.api.set_map(self.uniqueId, self.request)
                    print odoo.api.clients_map
                    # 启动Mqtt
                    if self.mqttdoor is None:
                        self.mqttdoor = MqttDoor(self.request, self.uniqueId)
                        self.mqttdoor.start()
                    # 启动心跳检测
                    if self.health is None:
                        self.health = HealthCheck(self.request, self.uniqueId)
                        self.health.start()
                elif action == 'getGoods':
                    self.uniqueId = receiveData['head']['uniqueId']
                    if self.uniqueId == '':
                        print 'uniqueId None'
                    else:
                        body = json.loads(receiveData['body'])
                        print body
                        key = "code"
                        val = ""
                        if ("code" in body) :
                            val = body["code"]
                            key = "barcode"
                        elif ("mac_id" in body):
                            val = body["mac_id"]
                            key = "mac"
                        elif ("gid" in body):
                            val = body["gid"]
                            key = "id"
                        elif ("gname" in body):
                            val = body["gname"]
                            key = "gname"
                        elif ("rfids" in body):
                            val = body["rfids"]
                            key = "rfids"
                                
                        if key == "mac":
                            data_list = getMacGoodsList(key, val)
                        elif key == "gname":
                            data_list = getGoodsListByName(key, val)
                        elif key == "rfids":
                            data_list = getRfidGoodsList(key, val)
                        else:
                            data_list = getGoodsList(key, val)
                        
                        returnGoods = {'exception': '', 'goods': data_list}
                        
                        if key == "mac":
                            data = json.dumps(returnGoods, encoding='gbk', ensure_ascii=False)
                            print data
                            odoo.api.send_data(self.request, data, self.uniqueId, 'getGoodsReturn', 'gbk')
                        else:
                            data = json.dumps(returnGoods)
                            print data
                            odoo.api.send_data(self.request, data, self.uniqueId, 'getGoodsReturn')
                elif action == 'getAllGoods':
                    if self.uniqueId != '':
                        body = json.loads(receiveData['body'])
                        if ("ost" in body) :
                            ost = body["ost"]
                        else:
                            ost = 0
                            
                        if ("lmt" in body) :
                            lmt = body["lmt"]
                        else:
                            lmt = 150
                            
                        data_list = getAllGoodsList(ost, lmt)
                        returnGoods = {'exception': '', 'goods': data_list}
                        data = json.dumps(returnGoods)
                        print data
                        odoo.api.send_data(self.request, data, self.uniqueId, 'getAllGoodsReturn')
                elif action == 'stockGoods':
                    if self.uniqueId != '':
                        body = json.loads(receiveData['body'])
                        print 'stockGoods'
                        picking_id = stockGoods(body)
                        returnData = {'exception': '','picking_id':str(picking_id)}
                        data = json.dumps(returnData)
                        print data
                        odoo.api.send_data(self.request, data, self.uniqueId, 'stockGoodsReturn')
                elif action == 'makePayment':
                    if self.uniqueId != '':
                        body = json.loads(receiveData['body'])
                        oid = body["oid"]
                        print 'makePayment'
                        #setOrderPay(oid)
                        returnData = {'exception': ''}
                        data = json.dumps(returnData)
                        print data
                        odoo.api.send_data(self.request, data, self.uniqueId, 'makePaymentReturn')
                elif action == 'getWarehouseGoods':
                    if self.uniqueId != '':
                        body = json.loads(receiveData['body'])
                        print 'getWarehouseGoods'
                        whid = body["whid"]
                        data_list = getWarehouseGoods(0, whid)
                        returnGoods = {'exception': '', 'goods': data_list}
                        data = json.dumps(returnGoods)
                        print data
                        odoo.api.send_data(self.request, data, self.uniqueId, 'getWarehouseGoodsReturn')
                        
                elif action == 'health':
                    uniqueId = receiveData['head']['uniqueId']
                    if self.uniqueId != '':
                        odoo.api.send_data(self.request, '', self.uniqueId, 'healthReturn')
                elif action == 'getPayInfo':
                    uniqueId = receiveData['head']['uniqueId']
                    if self.uniqueId != '':
                        # 生成支付订单
                        body = json.loads(receiveData['body'])
                        # 终端生成的order_id
                        order_id = body['order_id']
                        if uniqueId.startswith('DZC_'):
                            order = createDzcOrder(body)
                        else :
                            order = createOrder(body)
                            
                        if isinstance(order, str): 
                            returnUrl = {'exception': order, 'url': ''}
                            odoo.api.send_data(self.request, data, self.uniqueId, 'getPayInfoReturn')
                        elif order:
                            url = order['sendCode']
                            order_id = order['order_id']
                            amount_total = order['amount_total']
                            returnUrl = {'exception': '', 'url': url}
                            data = json.dumps(returnUrl)
                            #print data
                            #发送给java服务端生成订单
                            #json
                            #s = json.dumps({'orderCode':13, 'price':1, 'openid':'oJHHm5cg5ZVcDQ_qjjwl89Trro44', 'shopId':363})
                            #r = requests.post(url, data=s)
                            #parameter order_id':order[0] , 'sendCode': sendCode, 'amount_total
                             
                            if self.mqttdoor :
                                openId = self.mqttdoor.openId
                                _logger.info("openId :%s", openId)
                                post_datas = {'orderCode':str(order_id), 'price':str(amount_total), 'openid':openId, 'uniqueId':self.uniqueId}
                                r = requests.post(JAVA_SERVER_URL,data=post_datas)

                                print (r.text)
                                _logger.info("Create order url: %s result :%s", JAVA_SERVER_URL,r.text)
                            
                            odoo.api.send_data(self.request, data, self.uniqueId, 'getPayInfoReturn')
                            # 启动Mqtt
                            printList = order['printList']
                            if self.mqttCheck is None:
                                self.mqttCheck = MqttCheck(self.request, self.uniqueId, order_id, amount_total, printList)
                                self.mqttCheck.start()
                elif action == 'healthReturn':
                    uniqueId = receiveData['head']['uniqueId']
                    if self.health :
                        receive_time = time.time()
                        self.health.receive_time(receive_time)
                        self.health.check_hart(uniqueId)
                else:
                    continue
            except Exception, e:
                # print 'e.message:\t', e.message
                print 'traceback.print_exc():';traceback.print_exc()
                _logger.info("Exception: %s", traceback.format_exc())
                # print 'traceback.format_exc():\n%s' % traceback.format_exc()
                break
        self.request.close()
        if self.mqttdoor:
            self.mqttdoor.client_close()
            self.mqttdoor = None
        
        if self.health:
            self.health.client_close()
            self.health = None
        
        if self.mqttCheck:
            self.mqttCheck.client_close()
            self.mqttCheck = None
            
        odoo.api.del_map(self.uniqueId)
        print odoo.api.clients_map
        setLogout(self.uniqueId)
        print('%s use %d closed!' % (address, pid))
        _logger.info("%s use %d closed!", address ,pid)
