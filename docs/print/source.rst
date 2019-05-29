.. _source:

实现原理
============================

基本原理
----------------------------

本打印机按照标准打印机进行设计实现。通过接收并解析由打印控制命令和打印字符组成的字节串，
将字符以控制命令指定的方式放入framebuffer，然后控制打印机读取打印实现。

而在这里打印机连接的是blurr板，接收的是json格式的数据，因此再解析之前，首先对该接收数据进行封装，
成为标准字节串后，才将其传给标准打印机进行解析和打印。

打印命令
----------------------------

- LF    打印并换行
- CR    回车打印
- FF    换页
- ESC N 设置装订长
- ESC O 取消装订长
- ESC B 设置垂直造表指
- VT    执行垂直造表
- ESC D 设置水平造表值
- HT    执行水平造表
- SO    设置倍宽打印方式
- DC4   取消倍宽打印方式
- ESC A 设置字符行间距为n点行
- ESC @ 打印机初始化

硬件自检
----------------------------

硬件自检中，应有压杆、缺纸、温度、切刀位置检查四项。
但由于只有压杆可用，因此硬件自检暂时只检查了压杆是否抬起，
并对其增加了状态指示灯（红色）的显示。


代码讲解
============================

标准打印机类
----------------------------

- 构造函数
::

    def __init__(self):
        # 布局
        self.pageW = 864  # 页宽
        self.pageH = 256  # 页高
        self.wordW = 16  # 字符宽
        self.wordH = 16  # 字符高

        self.rowByte = int(self.pageW / 8)  # 字符宽所占字节
        self.rowWord = int(self.pageW / self.wordW)  # 每行字节数
        self.colWord = int(self.pageH / self.wordH)  # 每行字符数

        self.pageBuf = bytearray(int(self.pageW / 8 * self.pageH))
        self.pageFbuf = framebuf.FrameBuffer(self.pageBuf, self.pageW, self.pageH, framebuf.MONO_HLSB)

        # 属性
        self.dwDegree = 1  # 倍宽 初始为1倍
        self.roDegree = 0  # 旋转 初始为0度

        self.rowSpacing = 16  # 行间距为16点行
        self.bindingLength = 6  # 装订长为6行

        # 造表
        self.listVT = []  # 垂直造表列表
        self.listHT = []  # 水平造表列表

        # 定位坐标
        self.wordX = 0
        self.wordY = 0

        # 设置命令
        self._ESC = 0x1B
        self._HT = 0x09
        self._VT = 0x0B
        self._FF = 0x0C
        self._SO = 0x0E
        self._DC4 = 0x14
        self._FS = 0x1C
        self._LF = 0x0A
        self._CR = 0x0D

        # LED
        self.ledRED = Pin(Pin.cpu.A13, Pin.OUT)  # 状态指示灯
        self.ledGRN = Pin(Pin.cpu.A14, Pin.OUT)  # 电源指示灯

        # motor
        self.motorEn = Pin('Y7', Pin.OUT_PP)
        self.motorDr = Pin('Y5', Pin.OUT_PP)
        self.motorStep = Pin('Y6', Pin.OUT_PP)

        # printer LP2442
        # DST1 heat element number 1-432 Dots/DST 432
        # DST2 heat element number 432-864 Dots/DST 432
        # CLK  |`````|_____|`````|_____|`````|_____|````|
        # DAT  ZZZZZZZZZZXXXXZZZZZZZZZZZZZZZZZZZZZZZZZZZZ
        # LAT  ``````````````|_____|`````````````````````
        # DST  ```````````````````````|___|``````````````
        self.prtDST2 = Pin('X6', Pin.OUT_PP)
        self.prtDST1 = Pin('X7', Pin.OUT_PP)
        self.prtLATCH = Pin('X8', Pin.OUT_PP)
        self.prtCLK = Pin('Y9', Pin.OUT_PP)
        self.prtDAT = Pin('Y10', Pin.OUT_PP)
        self.prtTH_SW = Pin('X3', Pin.OUT_PP)

        self.deviceInit()

- 硬件初始化函数。初始化步进电机和打印机，并设置走纸键，按下usr键后走纸指定点行。
::

    # 硬件初始化
    def deviceInit(self):
        self.motorEn.value(1)
        self.motorDr.value(1)

        self.prtTH_SW.value(1)
        self.prtDST1.value(1)
        self.prtDST2.value(1)
        self.prtLATCH.value(1)

        # 走纸键
        USR_SW = pyb.Switch()
        USR_SW.callback(lambda: self.motorStepTo(160))

- 数据初始化函数。framebuffer清空，字符属性、造表列表、字符坐标复位。
::

    # 数据初始化
    def dataInit(self):
        self.pageFbuf.fill(0)

        # 属性
        self.dwDegree = 1
        self.roDegree = 0
        self.rowSpacing = 16
        self.bindingLength = 10

        # 造表
        self.listVT = []
        self.listHT = []

        # 定位坐标
        self.wordX = 0
        self.wordY = 0

- 硬件自检函数。暂时只检查了压杆是否抬起。
::

    # 硬件自检
    def deviceCheck(self):
        devBar = Pin('X1', Pin.IN)  # 压杆
        if devBar.value() == 0:
            bit2 = 0
        else:
            bit2 = 1
        # 返回错误数值
        err = bit2
        return err

- 控制步进电机转动函数。
::

    # 控制电机行进指定步数
    def motorStepTo(self, num):
        self.motorEn.value(0)
        for n in range(num):
            self.motorStep.value(0)
            self.motorStep.value(1)
            time.sleep_ms(1)
        self.motorEn.value(1)

- 打印framebuffer中指定的一行字符，共self.wordH个点行。
::

    # 打印一行
    def printRow(self, rowNum):
        # 定位到起始点行
        rowLine = rowNum * self.wordH
        # 打印每一点行
        self.motorEn.value(0)
        for li in range(self.wordH):
            # 输入数据
            for bi in range(self.rowByte):
                byte = self.pageBuf[(rowLine + li) * self.rowByte + bi]
                for bit in range(8):
                    self.prtCLK.value(0)
                    if byte & (128 >> bit):
                        self.prtDAT.value(1)
                    else:
                        self.prtDAT.value(0)
                    self.prtCLK.value(1)
            # 锁存数据
            self.prtLATCH.value(0)
            self.prtLATCH.value(1)
            # 使能加热供电
            self.prtTH_SW.value(0)
            self.prtTH_SW.value(1)
            # 使能加热
            self.prtDST1.value(0)
            self.prtDST2.value(0)
            time.sleep_ms(3)  # 加热2ms,试验发现低于2ms打印不清楚
            self.prtDST1.value(1)
            self.prtDST2.value(1)

            # 走纸
            self.motorStep.value(0)
            self.motorStep.value(1)
            time.sleep_ms(1)

        self.motorEn.value(1)

- 字符倍宽函数。将传进的正常字宽字符framebuffer加宽一倍后返回。
::

    # 字符倍宽
    def doubleWidth(self, wordFbuf):
        buf = bytearray(int(self.wordW / 8 * self.wordH * 2))
        dwWordFbuf = framebuf.FrameBuffer(buf, self.wordW * 2, self.wordH, framebuf.MONO_HLSB)
        for i in range(self.wordW):
            for j in range(self.wordH):
                dwWordFbuf.pixel(2 * i, j, wordFbuf.pixel(i, j))
                dwWordFbuf.pixel(2 * i + 1, j, wordFbuf.pixel(i, j))
        return dwWordFbuf

- 旋转字符函数。将传进的未旋转字符framebuffer旋转指定角度后返回。
::

    # 旋转字符
    def rotateWord(self, wordFbuf, roDegree):
        buf = bytearray(int(self.wordW / 8 * self.wordH))
        roWordFbuf = framebuf.FrameBuffer(buf, self.wordW, self.wordH, framebuf.MONO_HLSB)
        # n=1,90度
        if roDegree == 1:
            for i in range(self.wordW):
                for j in range(self.wordH):
                    roWordFbuf.pixel(j, i, wordFbuf.pixel(i, j))
        # n=2,180度
        elif roDegree == 2:
            for i in range(self.wordW):
                for j in range(self.wordH):
                    roWordFbuf.pixel(self.wordW - i, self.wordH - j, wordFbuf.pixel(i, j))
        # n=3,270度
        elif roDegree == 3:
            for i in range(self.wordW):
                for j in range(self.wordH):
                    roWordFbuf.pixel(self.wordH - j, i, wordFbuf.pixel(i, j))
        else:
            print('Rotation data error')
        return roWordFbuf

- 接收打印字节串，先硬件自检，如果无异常将字节串传给解析函数开始打印。
::

    # 硬件自检并打印
    def printData(self, data):
        err = self.deviceCheck()
        if err != 0:
            print('ERROR ' + str(err))
            print('Please put down the bar')
            self.ledRED.value(1)
            time.sleep(0.2)
            self.ledRED.value(0)
            time.sleep(1.2)
        else:
            try:
                self.analyseAndPrint(data)
            except:
                print('Data parsing error')

- 解析数据函数。解析数据，区分打印控制命令和打印字符，放入framebuffer，CR命令时打印。
::

    # 解析数据并打印
    def analyseAndPrint(self, data):
        length = len(data)
        bi = 0
        while bi < length:
            # ESC
            if data[bi] == self._ESC:
                # ESC @  初始化
                if data[bi + 1] == 0x40:
                    print("Init...")
                    self.deviceInit()
                    self.dataInit()
                    bi = bi + 2
                # ESC A n  设置行间距为n点行
                elif data[bi + 1] == 0x41:
                    n = data[bi + 2]
                    self.rowSpacing = n
                    bi = bi + 3
                # ESC N n  设置装订长为n行
                elif data[bi + 1] == 0x4E:
                    n = data[bi + 2]
                    self.bindingLength = n
                    print('Binding Length:' + str(self.bindingLength))
                    bi = bi + 3
                # ESC O  设置装订长为0行
                elif data[bi + 1] == 0x4F:
                    self.bindingLength = 0
                    print('Binding Length:' + str(self.bindingLength))
                    bi = bi + 2
                # ESC D  水平造表
                elif data[bi + 1] == 0x44:
                    bi = bi + 2
                    if data[bi] == 0:
                        self.listHT = []
                    else:
                        while data[bi] != 0:
                            if data[bi] > 0 and data[bi] <= self.rowWord:
                                self.listHT.append(data[bi])
                            else:
                                print('Invalid HT: ' + str(data[bi]))
                            bi = bi + 1
                    bi = bi + 1
                # ESC B  垂直造表
                elif data[bi + 1] == 0x42:
                    bi = bi + 2
                    if data[bi] == 0:
                        self.listVT = []
                    else:
                        while data[bi] != 0:
                            if data[bi] > 0 and data[bi] <= self.colWord:
                                self.listVT.append(data[bi])
                            else:
                                print('Invalid VT: ' + str(data[bi]))
                            bi = bi + 1
                    bi = bi + 1

            # HT  执行水平造表
            elif data[bi] == self._HT:
                if len(self.listHT) != 0:
                    xi = self.listHT.pop(0)
                    self.wordX = xi - 1
                else:
                    pass
                bi = bi + 1

            # VT  执行垂直造表
            elif data[bi] == self._VT:
                if len(self.listVT) != 0:
                    yi = self.listVT.pop(0)
                    if self.wordY < yi:
                        self.wordY = yi - 1
                        self.wordX = 0
                else:
                    pass
                bi = bi + 1

            # FF  换页
            elif data[bi] == self._FF:
                num = int(self.bindingLength * self.wordH)
                self.motorStepTo(num)
                bi = bi + 1

            # SO  字符倍宽
            elif data[bi] == self._SO:
                print("Set double width")
                self.dwDegree = 2
                bi = bi + 1

            # DC4  取消字符倍宽
            elif data[bi] == self._DC4:
                print("Cancel double width")
                self.dwDegree = 1
                bi = bi + 1

            # FS 2 n  旋转字符
            elif data[bi] == self._FS:
                if data[bi + 1] == 0x49:
                    print("Rotate the word")
                    self.roDegree = data[bi + 2]
                    bi = bi + 3

            # LF  打印一行
            elif data[bi] == self._LF:
                print("Printing the row...")
                self.printRow(self.wordY)
                self.motorStepTo(self.rowSpacing)
                self.wordX = 0
                self.wordY = self.wordY + 1
                bi = bi + 1

            # CR  回车打印全部
            elif data[bi] == self._CR:
                print("Printing...")
                for i in range(self.wordY + 1):
                    self.printRow(i)
                    self.motorStepTo(self.rowSpacing)
                self.wordX = 0
                self.wordY = 0
                self.pageFbuf.fill(0)
                bi = bi + 1

            # 如果是字符，放入framebuffer
            else:
                if data[bi] >= 0x00 and data[bi] <= 0x7F:
                    bytes = 1
                elif data[bi] >= 0xC0 and data[bi] <= 0xDF:
                    bytes = 2
                elif data[bi] >= 0xE0 and data[bi] <= 0xEF:
                    bytes = 3
                elif data[bi] >= 0xF0 and data[bi] <= 0xF7:
                    bytes = 4
                elif data[bi] >= 0xF8 and data[bi] <= 0xFB:
                    bytes = 5
                elif data[bi] >= 0xFC and data[bi] <= 0xFF:
                    bytes = 6
                else:
                    print('Invalid command: ' + str(hex(data[bi])))
                    bi = bi + 1
                    continue

                # 解码字符
                wordBuf = data[bi: bi + bytes]
                word = wordBuf.decode('utf-8')

                # 正常字符
                if self.dwDegree == 1 and self.roDegree == 0:
                    self.pageFbuf.text(word, self.wordX * self.wordH, self.wordY * self.wordW)
                # 字符旋转与倍宽
                else:
                    buf = bytearray(int(self.wordW / 8 * self.wordH))
                    wordFbuf = framebuf.FrameBuffer(buf, self.wordW, self.wordH, framebuf.MONO_HLSB)
                    wordFbuf.text(word, 0, 0)
                    if self.roDegree != 0:
                        wordFbuf = self.rotateWord(wordFbuf, self.roDegree)
                        self.roDegree = 0
                    if self.dwDegree == 2:
                        wordFbuf = self.doubleWidth(wordFbuf)
                    self.pageFbuf.blit(wordFbuf, self.wordX * self.wordH, self.wordY * self.wordW)

                # framebuffer坐标重定位
                if self.wordX < self.rowWord - self.dwDegree:
                    self.wordX = self.wordX + self.dwDegree
                else:
                    self.wordX = 0
                    self.wordY = self.wordY + 1

                # 取下一个字节
                bi = bi + bytes

数据封装类
----------------------------

- 构造函数。定义商品小票标题和行标名称。
::

    def __init__(self):
        self.title = '新零售标准店铺'
        self.tr1Td1 = '商品'
        self.tr1Td2 = '数量×单价'
        self.tr1Td3 = '金额'
        self.tr2Td1 = '合计：'
        self.tr2Td2 = '元'

- 封装函数。根据需要的排版定义打印控制命令，将它和接收的字典中的打印字符组装成标准打印机可识别的字节串并返回。
::

    def encodeToPrint(self, dataDict):
        productList = dataDict['productList']
        totalAmount = dataDict['totalAmount']

        # 分割线
        printCom0 = '\x1B\x44\x07\x00'  # 水平造表，水平位置为0x07
        printLine = printCom0 + '\x09' + '-----------------------------------------' + '\x0D'  # 0x09执行水平造表，0x0D换行并打印

        # 标题
        printCom1 = '\x1B\x44\x14\x00'  # 水平造表
        printData1 = printCom1 + '\x09' + '\x0E' + self.title + '\x14' + '\x0D'

        # 表头
        printCom2 = '\x1B\x44\x07\x1E\x2A\x00'  # 水平造表，水平位置分别为0x07,0x1E,0x2A
        printData2 = printCom2 + '\x09' + self.tr1Td1 + '\x09' + self.tr1Td2 + '\x09' + self.tr1Td3 + '\x0D'

        # 表项
        printCom3 = '\x1B\x44\x07\x1E\x2A\x00'  # 水平造表
        printData3 = ''
        for li in productList:
            printData3 = printData3 + printCom3 + '\x09' + li['name'] + '\x09' + li['count'] + '×' + li[
                'price'] + '\x09' + li['total'] + '\x0D'

        # 表尾
        printCom4 = '\x1B\x44\x07\x00'  # 水平造表
        printData4 = printCom4 + '\x09' + self.tr2Td1 + totalAmount + self.tr2Td2 + '\x0D'

        # 返回封装数据
        printData = printLine + printData1 + printLine + printData2 + printLine + printData3 + printLine + printData4 + printLine
        printData = printData + '\x0C'  # 0x0C换到下一页
        return printData.encode('utf-8')

主函数
----------------------------

先将uart接收到的json数据还原为字典形式，然后对其进行封装，封装完成后传给标准打印机进行解析打印。
::

    if __name__ == '__main__':
        uart = UART(4, baudrate=115200, bits=8, parity=None, stop=1)
        gnd = Pin('X4', Pin.OUT_PP)  # 定义X4模拟uart的GND引脚
        gnd.value(0)
        psd = PackShopData()
        pr = Printer()
        while True:
            if uart.any():
                uartData = uart.read()
                uartData = uartData.decode('utf-8')
                try:
                    dataDict = json.loads(uartData)
                    data = psd.encodeToPrint(dataDict)
                    pr.printData(data)
                except:
                    print("Data is illegal and cannot be printed")

