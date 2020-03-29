# libmodbus-windows
libmodbus windows 使用说明

## modbus-tcp 网络结构
面向TCP可靠连接，由tcp client端主动发起消息，TCP Server端被动响应，应答模式。
一般采用一个client对应多个server或者一个client对应一个server网络结构。

- modbus client 发起方，在实际应用中的地位可能是服务器（或PC电脑）。一般存在一个。
- Modbus Server 接收响应方，在实际应用中的地位可能是子设备（嵌入式外部设备）。存在多个。

## libmodbus 开源modbus库
- modbus 协议资料：[www.modbus.org](http://www.modbus.org)
- The official website is [www.libmodbus.org](http://www.libmodbus.org).


## windows 安装（编译）

### 推荐vcpkg安装

```cmd
vcpkg install libmodbus

//安装64位的
vcpkg install libmodbus:x64-windows
```

### 手动下载编译
下载源码后打开libmodbus/src/win32/README.win32按照说明：

1. 在libmodbus/src/win32目录下执行 configure.js（双击或者命令，详见README.win32）
2. 打开modbus-9.sln
3. 编译（注意是32位编译）
4. 拷贝modbus.dll modbus.lib 到需要用的工程目录下。
5. 项目工程中添加libmodbus/src到include目录中，dll也需要注册（或者配置库目录）。


*简言之：繁琐！*



### 故障码errno
window下libmodbus 无法正常获取modbus_strerror(errno)
所有的errno都返回的是“Unknow Error"，需要手动编写一个转换函数。

```c++
//采用仿函数书写方法
struct ModbusError {
    inline std::string  operator()(const int& err)
    {
        switch (err) {
        case EMBXILFUN:
            return "Illegal function";
        case EMBXILADD:
            return "Illegal data address";
        case EMBXILVAL:
            return "Illegal data value";
        case EMBXSFAIL:
            return "Slave device or server failure";
        case EMBXACK:
            return "Acknowledge";
        case EMBXSBUSY:
            return "Slave device or server is busy";
        case EMBXNACK:
            return "Negative acknowledge";
        case EMBXMEMPAR:
            return "Memory parity error";
        case EMBXGPATH:
            return "Gateway path unavailable";
        case EMBXGTAR:
            return "Target device failed to respond";
        case EMBBADCRC:
            return "Invalid CRC";
        case EMBBADDATA:
            return "Invalid data";
        case EMBBADEXC:
            return "Invalid exception code";
        case EMBMDATA:
            return "Too many data";
        case EMBBADSLAVE:
            return "Response not from requested slave";
        case ETIMEDOUT:
            return "Time out";
        case EINVAL:
            return "Invalid";
        case ECONNRESET:
            return "Connect Reset";
        case EBADF:
            return "Bad File";
        default:
            return "Unknow Error("+std::to_string(err)+")";
        }
    }
};

//使用方法
std::string errStr=ModbusError()(errno);
```


## Modbus-TCP协议格式
*ModbusTcp 因为采用的是TCP可靠连接，链路保证了数据可靠，因此没有校验字节*
### 客户端（modbus client)
|事务标识符|协议标识符|长度|单元标识符（客户端ID）|功能码|起点地址|寄存器数量|
|--|--|--|--|--|--|--|
|2字节|2字节|2字节|1字节|1字节|2字节|2字节|

实例：
```
00 01 00 00 00 06 01 03 00 0D 00 01 
- 事务标识符 00 01
- 协议标识符 00 00
- 长度      00 06
- 单元标识符（终端ID） 01
- 功能码 03
- 起点地址 00 0D
- 数据个数 00 01
```

### 服务器（modbus server）

|事务标识符|协议标识符|长度|单元标识符（客户端ID）|功能码|数据长度（字节数）|数据|
|--|--|--|--|--|--|--|
|2字节|2字节|2字节|1字节|1字节|1字节|N字节|


### libmodbus  Modbus-TCP使用代码
1. 创建modbus设备
```code
	modbus_ = modbus_new_tcp(ip_.c_str(), 502);
```
2. 设置参数
*必须设置modbus slave参数，否则默认可能是0xff，无效ID。从而导致发送数据服务器不响应（无效ID服务器直接丢弃。）*
```code
//modbus_set_debug(modbus_, TRUE);
		//设置响应超时时间1000ms
		modbus_set_response_timeout(modbus_, 0, 1000 * 1000);
		//modbus_set_error_recovery(modbus_, modbus_error_recovery_mode::MODBUS_ERROR_RECOVERY_LINK);
		modbus_set_slave(modbus_, 1);
```
3. 连接服务器
```code

		int ret = modbus_connect(modbus_);
```
4. 读写操作
*modbus读写单位为16位，寄存器个数\*2=总字节数*
```code 

	uint16_t status;
	int ret = modbus_read_registers(modbus_, ADDR_NAV_STATUS, 1, &status);
    
	int len = sizeof(point) / (sizeof(uint16_t));
	int ret = modbus_write_registers(modbus_, ADDR_POINT_PARAM, len, (uint16_t*)(&point));
```
5. modbus采用TCP长连接，服务器异常断线客户端需要重新连接
- modbus服务器（modbus server，可能是某个modbus子设备）
- 如果modbus server异常断线，client可能无法感知到，此时client发送的所有请求都可能暂存在网络链路途中（中继、路由）。无法读取数据，读写操作的返回值为ETIMEOUT。
- 即使服务器重启或重新恢复了网络，但是与modbus client 之间的连接socket已经断开，需要client重新建立连接。因此client需要支持断线检测，自动重连。