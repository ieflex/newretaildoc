.. _source:

实现原理
============================

基本原理
----------------------------

ZigBee模块入网时，通过网关请求服务器，接收到服务器的消息后，将AT指令通过uart发送给TB开发板。
TB解析收到的AT指令，得到商品价签信息，然后驱动lcd模块将价签信息显示在屏幕上，

字库说明
----------------------------

由于TB接收到的商品信息数据是gbk编码的（MicroPython只支持utf8编码）且包含汉字，
因此我们根据需要制作了自己的字库，包含大部分ASCII字符和一些用到的汉字，
基本上涵盖了商品信息所用到的字符，如仍有未在字库的字符出现，将暂时以空格代替。

屏幕驱动
----------------------------

根据以上内容，我们编写了lcd的屏幕驱动，使其能显示字符并支持画线。
其中传入的字符规定为bytes类型，通过查询字库直接获得点阵信息。


代码讲解
============================

屏幕显示类
----------------------------

- 构造函数。定义SPI以驱动lcd，定义用到的商品信息条目字符的gbk编码。
::

    class LcdShow():
        def __init__(self):
            self.usrspi = USR_SPI(scl=Pin('Y6', Pin.OUT_PP), sda=Pin('Y7', Pin.OUT), dc=Pin('Y8', Pin.OUT))
            self.display = DISPLAY(self.usrspi, cs=Pin('Y5', Pin.OUT), res=Pin('Y4', Pin.OUT), led_en=Pin('Y3', Pin.OUT))
            self.itemNameList = [b'\xb2\xfa\xb5\xd8\xa3\xba',  # 品名：
                                 b'\xd7\xd6\xba\xc5\xa3\xba',  # 单价：
                                 b'\xcc\xd8\xc9\xab\xa3\xba',  # 特色：
                                 b'\xb5\xa5\xbc\xdb\xa3\xba',  # 字号：
                                 b'\xc6\xb7\xc3\xfb\xa3\xba']  # 产地：

- 显示函数。先清空屏幕，然后画出显示框，最后显示出商品价签信息。
::

    def lcdDisplay(self, itemInfoList):
        print('clear the screen')
        self.display.clr(self.display.WHITE)
        self.display.put_hline(2, 2, 124, self.display.BLACK)
        self.display.put_vline(2, 2, 156, self.display.BLACK)
        self.display.put_hline(2, 157, 124, self.display.BLACK)
        self.display.put_vline(125, 2, 156, self.display.BLACK)
        x = 4
        y = 4
        print('display the commodity information')
        for i in range(5):
            bytes = self.itemNameList[4 - i] + itemInfoList[4 - i]
            self.display.putstr(x, y, bytes, self.display.BLACK)
            y = y + 18

AT指令解析类
----------------------------

- 构造函数。定义商品价签信息列表，并初始化为空。
::

    class ATrev():
        def __init__(self):
            self.itemInfoList = []

- 解析商品信息部分数据，并将各个条目信息放入列表中。
::

    def analyseData(self, message):
        totalLength = len(message)
        bi = 0
        while bi < totalLength:
            byteCount = message[bi]
            bi = bi + 1
            charCount = message[bi]
            bi = bi + 1
            itemInfo = message[bi: bi + byteCount]
            self.itemInfoList.append(itemInfo)
            bi = bi + byteCount

- 解析数据包，获取商品价签信息。
::

    def unpackData(self, data):
        totalLength = len(data)
        bi = 0
        while bi < totalLength:
            # 读取包头
            start = data[bi]
            if start == 0x7E:
                bi = bi + 1
                # 读取数据长度
                dataLength = data[bi: bi + 2]
                dataLength = struct.unpack('>H', dataLength)
                dataLength = dataLength[0] - 5
                bi = bi + 2
                type = data[bi]
                # 读取帧类型
                if type == 0x88:
                    bi = bi + 1
                    fid = data[bi]
                    bi = bi + 1
                    ATcom = data[bi: bi + 2]
                    bi = bi + 2
                    ATok = data[bi]
                    bi = bi + 1
                    message = data[bi: bi + dataLength]
                    bi = bi + dataLength
                    # 读取包尾
                    ending = data[bi]
                    if ending == 0xF0:
                        print("succeed to get commodity information")
                        self.analyseData(message)
                        iil = self.itemInfoList
                        self.itemInfoList = []
                        return iil
                    else:
                        print("can't find the ending of the frame")
                        return []
                else:
                    print("frame in wrong type")
                    return []
            else:
                print("can't find the start of the frame")
                return []

主函数
----------------------------

先对uart接收到的数据进行解析，获得商品价签信息列表，然后控制lcd进行显示。
::

    if __name__ == '__main__':
        uart = UART(4, baudrate=115200, bits=8, parity=None, stop=1)
        atr = ATrev()
        lcd = LcdShow()
        while True:
            if uart.any():
                uartdata = uart.read()
                print("receive a frame:")
                print(ubinascii.hexlify(uartdata, ' '))
                itemInfoList = atr.unpackData(uartdata)
                if len(itemInfoList) == 5:
                    lcd.lcdDisplay(itemInfoList)
                else:
                    print('incomplete data')
