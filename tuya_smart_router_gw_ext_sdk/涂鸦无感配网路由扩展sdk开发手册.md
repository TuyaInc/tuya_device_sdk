## sdk能力介绍

sdk具备两大能力，设备接入和路由器管理。

- 支持涂鸦所有具有无感配网能力的IOT设备接入

- 支持在涂鸦智能APP端：

    开启/关闭2.4G和5G WIFI；

    修改账号密码；

    选择信号强度；

    配置加密方式；

    实时显示接入路由器的在线设备，以及限速功能

## pegasus配网

### 无感配网概述

涂鸦无感配网方案采用的是802.11协议中Beacon/Probe Request/Probe Respone帧实现待配网设备闪电配网，接入涂鸦云。同时涂鸦闪电配网方案可做到同时支持未配网设备配网、已配网设备无感更换路由器。

已配网设备在已进入可添加设备状态后，广播Beacon包，待配网IoT设备接收含有闪电配网OUI标识的beacon包后，广播的含有设备信息的ProbeRequest数据包，将设备相关信息上报给周边具体涂鸦闪电配网能力的已配网设备。

已配网设备在已允许添加设备的情况下，收到设备广播的相关信息后，推送给云端，获取当前设备加密密钥，并下发至已配网设备。已配网设备通过Probe Respone帧，将配网信息加密后发送给智能终端设备，智能终端设备根据UUID及相关数据计算出秘钥，对加密的配网信息解密，连接到网络，并进一步在云端激活绑定。闪电配网方案采用一机一密云端动态生成密钥，生成配网数据，安全性高。（信息第一次采用固定秘钥加密）

### 名词解释

**角色**：

1. 已配网者(server)：已连接涂鸦云的IoT设备，并具有闪电配网服务提供者。可包括：路由器、WiFi网关、WiFi音箱及WiFi智能IoT设备
2. 待配网者(client)：待连接涂鸦云的IoT设备，并具有闪电配网能力。包括：WiFi网关、WiFi音箱及WiFi智能IoT设备
3. pegasus：是指闪电配网 or 无感配网的别称

### 技术要求

1. 待配网设备具有在应用层发送802.11协议中的Probe Request，接收Probe Respone帧、beacon帧的能力，并能处理VSIE中的特定OUI
2. 已配网设备具有在应用层接收802.11协议中的Probe Request、发送Probe Respone帧、beacon帧的能力，并能处理VSIE中的特定OUI

### 涂鸦oui

涂鸦闪电配网协议的OUI为0x68,0x57,0x2d。

## 开发说明

### 授权相关

在demo 中 PID，uuid & authkey 仅用作测试使用，不能用于实际产品，会导致后续产品不可用。PID 需要用户自行从涂鸦开发者平台申请。每一台设备都必须有自己uuid & authkey，且唯一。

### 编译说明

涂鸦根据客户toolchain编译成的相关的库，附带开发手册以及相关demo等提供给客户。



## sdk使用说明

### demo文件说明

| 文件名                    | 作用                              |
| ------------------------- | --------------------------------- |
| user_main.c               | sdk demo主入口文件                |
| tuya_iot_wired_net.c      | 有线网络配置，例如获取IP和MAC地址 |
| tuya_pegasus_test.c       | 无感配网服务相关                  |
| tuya_route_service_test.c | 路由器业务相关                    |



### 调用流程说明

![image-20200611203534116](../../../bakup/picture/markdown/image-20200611203534116.png)

详细见user_main.c文件

```c
//授权信息写入、设置存储路径
	memset(&env, 0 , sizeof(TUYA_GW_ENV_S));
    snprintf(env.storage_path, sizeof(env.storage_path), "%s", CFG_STORAGE_PATH);
    snprintf(env.product_key, sizeof(env.product_key), "%s", PRODUCT_KEY);
    snprintf(env.uuid, sizeof(env.uuid), "%s", UUID);
    snprintf(env.auth_key, sizeof(env.auth_key), "%s", AUTHKEY);
    snprintf(env.sw_ver, sizeof(env.sw_ver), "%s", USER_SW_VER);
    env.is_oem = TRUE;
//注册网关回调业务接口
    env.gw_stat_changed_cb = __gw_dev_status_changed_cb;
    env.gw_ug_inform_cb = __gw_dev_rev_upgrade_info_cb;
    env.gw_reset_cb = __gw_dev_reset_inform_cb;
    env.gw_reboot_cb = __gw_dev_reboot_cb;
    env.obj_dp_cmd_cb = __gw_dev_obj_dp_cmd_cb;
    env.raw_dp_cmd_cb = __gw_dev_raw_dp_cmd_cb;
    env.dp_query_cb = __gw_dev_dp_query_cb;
    env.nw_stat_cb = __gw_dev_net_status_cb;
    env.get_log_file = __gw_get_log_file_cb;
//开启网关服务，sdk接入涂鸦云
    op_ret = tuya_gw_service_start(&env);
    if (OPRT_OK != op_ret) {
        printf("tuya_gw_start error:%d\n", op_ret);
        return op_ret;
    }

//无感配网服务
    op_ret = tuya_pegasus_test_start();
    if(OPRT_OK != op_ret) {
        PR_ERR("tuya_pegasus_test_start err:%d",op_ret);
        return -5;
    }

//路由器业务
    op_ret = tuya_route_test_start();
    if(OPRT_OK != op_ret) {
        PR_ERR("tuya_route_test_start err:%d",op_ret);
        return -6;
    }
```



### 网关服务流程及接口

#### tuya_gw_service_start

```c
OPERATE_RET tuya_gw_service_start(TUYA_GW_ENV_S *p_env);

typedef struct {    
    CHAR_T storage_path[STR_LEN_MAX+1];		//sdk存储数据的路径，该路径必须对应一个可读写文件系统分区
    CHAR_T product_key[STR_LEN_MAX+1];		//产品id/key
    CHAR_T uuid[STR_LEN_MAX+1];				//授权信息，uuid
    CHAR_T auth_key[STR_LEN_MAX+1];			 //授权信息，authkey
    CHAR_T sw_ver[STR_LEN_MAX+1];		      //sdk版本号				
    BOOL_T is_oem; 							//是否为oem2.0

    TUYA_GW_STAT_CHANGED_CB gw_stat_changed_cb;	//设备云端状态变更回调
    TUYA_GW_UG_INFORM_CB    gw_ug_inform_cb;	//设备升级入口
    TUYA_GW_RESET_IFM_CB    gw_reset_cb;		//设备复位请求入口
    TUYA_GW_REBOOT_CB       gw_reboot_cb;		//设备进程重启请求入口
    TUYA_GW_OBJ_DP_CMD_CB   obj_dp_cmd_cb;		//设备格式化指令数据下发入口
    TUYA_GW_RAW_DP_CMD_CB   raw_dp_cmd_cb;		//设备透传指令数据下发入口
    TUYA_GW_DP_QUERY_CB     dp_query_cb;		//设备特定数据查询入口
    TUYA_GW_NW_STAT_CB      nw_stat_cb;			//外网状态变更回调
    TUYA_GW_GET_LOG_FILE_CB get_log_file;		//远程拉去sdk日志文件

#ifdef TUYA_ZIGBEE_ENABLE    
    TUYA_ZB_CONFIG_S zb_config;					//zigbee配置信息					
#endif    
} TUYA_GW_ENV_S;

#ifdef TUYA_ZIGBEE_ENABLE
typedef struct{
    CHAR_T serial_port[STR_LEN_MAX+1];    //zigbee串口设备号
    BOOL_T is_cts;                          //是否带有流控

    CHAR_T tmp_dir[STR_LEN_MAX+1];        //临时文件目录
    CHAR_T bin_dir[STR_LEN_MAX+1];        //bin文件目录，勿存放文件，其他平台可能为只读文件系统
    CHAR_T log_dir[STR_LEN_MAX+1];        //日志存储目录

} TUYA_ZB_CONFIG_S;
#endif
```



**函数描述**

开启网关服务，注册所需回调。

**参数**

| 输入/输出 | 参数名            | 描述                   |
| --------- | ----------------- | ---------------------- |
| [IN]      | p_cbs             | 无感配网回调结构体指针 |
| [IN]      | get_probe_intr_ms | 轮询probe帧的间隔      |

**返回值**

0 : 成功； 其他：失败错误码

**说明**

网关基础服务为涂鸦联网sdk，用户需要zigbee网关功能的话，可选择带有zigbee网关功能的路由sdk进行开发，无感配网路由sdk及其扩展sdk对外接口一致。



##### 注册回调gw_stat_changed_cb

```c
typedef VOID (*TUYA_GW_STAT_CHANGED_CB)(IN CONST GW_STATUS_E status);
//示例
VOID __gw_dev_status_changed_cb(IN CONST GW_STATUS_E status);
```

**函数描述**

当网关状态变化，通过此回调通知用户。

**参数**

| 输入/输出 | 参数名 | 描述                                                         |
| --------- | ------ | ------------------------------------------------------------ |
| [IN]      | status | 网关的状态：<br />0）网关重置。<br />1）网关已激活。<br />2）第一次启动。<br />3）网关已激活已开始运行。 |

**返回值**

无



##### 注册回调gw_ug_inform_cb

```c
typedef VOID (*TUYA_GW_UG_INFORM_CB)(IN CONST FW_UG_S *fw);;
//示例
VOID __gw_dev_rev_upgrade_info_cb(IN CONST FW_UG_S *fw);
```

**函数描述**

网关 OTA 固件通知，表示服务端有新版本的固件。

**参数**

| 输入/输出 | 参数名 | 描述                                                         |
| --------- | ------ | ------------------------------------------------------------ |
| [IN]      | fw     | 保存固件的类型，固件下载url地址，固件md5值，版本，以及文件大小。参见 tuya_cloud_com_defs.h 定义。 |

**返回值**

无

**说明**

用户根据该结构体信息，下载固件以及校验等。



##### 注册回调gw_reset_cb

```c
typedef VOID (*TUYA_GW_RESET_IFM_CB)(GW_RESET_TYPE_E type);
//示例
VOID __gw_dev_reset_inform_cb(GW_RESET_TYPE_E type);
```

**函数描述**

网关复位回调通知。网关应用需要重启。

**参数**

| 输入/输出 | 参数名 | 描述                                                         |
| --------- | ------ | ------------------------------------------------------------ |
| [IN]      | type   | 重置类型：<br />1）本地恢复工厂设置。<br />2）远端网关重置，即为通过APP取消与账号绑定。<br />3）本地网关重置，取消与账号绑定。<br />4）远端恢复工厂设置。<br />5）恢复数据工厂设置。 |

**返回值**

无

**说明**

该接口内容由sdk使用者实现。



##### 注册回调gw_reboot_cb

```c
typedef VOID (*TUYA_GW_REBOOT_CB)(VOID);
//示例
VOID __gw_dev_reboot_cb(VOID);
```

**函数描述**

网关复位重启。

**参数**

| 输入/输出 | 参数名 | 描述 |
| --------- | ------ | ---- |
|           |        |      |

**返回值**

无

**说明**

该接口内容由sdk使用者实现。



##### 注册回调obj_dp_cmd_cb

```c
typedef VOID (*TUYA_GW_OBJ_DP_CMD_CB)(IN CONST TY_RECV_OBJ_DP_S *dp);
//示例
VOID __gw_dev_obj_dp_cmd_cb(IN CONST TY_RECV_OBJ_DP_S *dp);
```

**函数描述**

OBJ 功能点信息命令回调。

**参数**

| 输入/输出 | 参数名 | 描述                                                         |
| --------- | ------ | ------------------------------------------------------------ |
| [IN]      | dp     | TY_RECV_OBJ_DP_S 中包含：<br />1）dp 命令类型。<br />2）dp点传输类型<br />3）dp 点控制的设备id<br />4）当dtt_tp 为多播时，则mb_id 群组id。<br />5）功能点结构体数组长度。<br />6）功能点结构体数组。<br />参见 tuya_cloud_com_defs.h |

**返回值**

无

**说明**





##### 注册回调raw_dp_cmd_cb

```c
typedef VOID (*TUYA_GW_RAW_DP_CMD_CB)(IN CONST TY_RECV_RAW_DP_S *dp);
//示例
VOID __gw_dev_raw_dp_cmd_cb(IN CONST TY_RECV_RAW_DP_S *dp);
```

**函数描述**

透传类功能点信息命令回调。

**参数**

| 输入/输出 | 参数名 | 描述                                                         |
| --------- | ------ | ------------------------------------------------------------ |
| [IN]      | mac    | 该结构体中包含：<br />1）dp 命令类型。<br />2）dp点传输类型<br />3）dp 点控制的设备id。<br />4）用户定义功能点序号<br />5）当dtt_tp 为多播时，则mb_id 群组id。<br />6）透传数据字节个数。<br />7）透传数据。<br />参见 tuya_cloud_com_defs.h |

**返回值**

无

**说明**





##### 注册回调dp_query_cb

```c
typedef VOID (*TUYA_GW_DP_QUERY_CB)(IN CONST TY_DP_QUERY_S *dp_qry);
//示例
VOID __gw_dev_dp_query_cb(IN CONST TY_DP_QUERY_S *dp_qry)
```

**函数描述**

设备特定数据查询入口。

**参数**

| 输入/输出 | 参数名 | 描述                                                         |
| --------- | ------ | ------------------------------------------------------------ |
| [IN]      | dp_qry | 该结构体中包含：<br />1）dp 点控制的设备id。<br />2）查询的dp个数。<br />3）用户定义功能点序号集。<br />参见 tuya_cloud_com_defs.h |

**返回值**

无

**说明**





##### 注册回调nw_stat_cb

```c
typedef VOID (*TUYA_GW_NW_STAT_CB)(IN CONST GW_BASE_NW_STAT_T stat);
//示例
OPERATE_RET ty_pegasus_get_mac_cb(OUT NW_MAC_S *mac);
```

**函数描述**

网络状态管理回调，用户可以根据网络状态，做自己的事务处理。。

**参数**

| 输入/输出 | 参数名   | 描述                                                         |
| --------- | -------- | ------------------------------------------------------------ |
| [IN]      | 网络状态 | GB_STAT_LAN_UNCONN: LAN没有连接<br />GB_STAT_LAN_CONN: LAN已连接但云端没连接。<br />GB_STAT_CLOUD_CONN: WAN已连接到云端 |

**返回值**

无

**说明**

为2的时候表示用户已经成功连接到云端了。



##### 注册回调get_log_file

```c
typedef VOID (*TUYA_GW_GET_LOG_FILE_CB)(OUT CHAR_T *file_name, IN CONST INT_T len);
//示例
VOID __gw_get_log_file_cb(OUT CHAR_T *file_name, IN CONST INT_T len);
```

**函数描述**

获取sdk日志。

**参数**

| 输入/输出 | 参数名    | 描述                           |
| --------- | --------- | ------------------------------ |
| [OUT]     | file_name | 日志压缩包文件名，带上绝对路径 |
| [IN]      | len       | 日志压缩包文件名长度           |

**返回值**

无

**说明**

用户可以根据自己设备的性能决定保存多久的日志，我们建议至少保存三天的sdk日志，便于远程技术支持以及问题定位。



#### tuya_gw_service_reset

```c
OPERATE_RET tuya_gw_service_reset(VOID)
```

**函数描述**

网关sdk复位，置为未激活状态。

**参数**

| 输入/输出 | 参数名 | 描述 |
| --------- | ------ | ---- |
|           |        |      |

**返回值**

无

**说明**

带zigbee功能的会禁止zigbee子设备接入并将网关置为未激活状态；

不带zigbee功能的将网关置为未激活状态。



### 无感配网流程及接口

#### tuya_hal_pegasus_init

```c
OPERATE_RET tuya_hal_pegasus_init(TUYA_PEGASUS_CBS_S *p_cbs, UINT_T get_probe_intr_ms);

typedef struct {
    TY_PEGASUS_EVENT_CB event_cb;
    TY_PEGASUS_SEND_FRAME_CB send_frame_cb;
    TY_PEGASUS_GET_SSID_PWD_CB get_ssid_pwd_cb;
    TY_PEGASUS_GET_MAC_CB get_mac_cb;
    TY_PEGASUS_SET_OUI_CB set_oui_cb;
    
} TUYA_PEGASUS_CBS_S;
```

**函数描述**

无感配网服务初始化函数，注册所需回调。

**参数**

| 输入/输出 | 参数名            | 描述                   |
| --------- | ----------------- | ---------------------- |
| [IN]      | p_cbs             | 无感配网回调结构体指针 |
| [IN]      | get_probe_intr_ms | 轮询probe帧的间隔      |

**返回值**

0 : 成功； 其他：失败错误码

**说明**

网关基础服务为涂鸦联网sdk，用户需要zigbee网关功能的话，可选择带有zigbee网关功能的路由sdk进行开发，无感配网路由sdk及其扩展sdk对外接口一致。



##### 注册回调get_mac_cb

```c
typedef OPERATE_RET (*TY_PEGASUS_GET_MAC_CB)(OUT NW_MAC_S *mac);
//示例
OPERATE_RET ty_pegasus_get_mac_cb(OUT NW_MAC_S *mac);
```

**函数描述**

获取ap的mac地址。

**参数**

| 输入/输出 | 参数名 | 描述          |
| --------- | ------ | ------------- |
| [OUT]     | mac    | 路由器mac地址 |

**返回值**

0 : 成功； 其他：失败错误码

**说明**

该接口内容由sdk使用者实现。



##### 注册回调get_ssid_pwd_cb

```c
typedef OPERATE_RET (*TY_PEGASUS_GET_SSID_PWD_CB)(OUT UINT8_T *ssid, IN INT_T slen, OUT UINT8_T *pwd, IN INT_T plen);
//示例
OPERATE_RET ty_pegasus_get_ssid_pwd_cb(OUT UINT8_T *ssid, IN INT_T slen, OUT UINT8_T *pwd, IN INT_T plen)
```

**函数描述**

获取2.4G WIFI的ssid和passwd。

**参数**

| 输入/输出 | 参数名 | 描述           |
| --------- | ------ | -------------- |
| [OUT]     | ssid   | ssid内存首地址 |
| [IN]      | slen   | ssid长度       |
| [OUT]     | pwd    | pwd内存首地址  |
| [IN]      | plen   | pwd长度        |

**返回值**

0 : 成功； 其他：失败错误码

**说明**

该接口内容由sdk使用者实现。



##### 注册回调set_oui_cb

```c
typedef OPERATE_RET (*TY_PEGASUS_SET_OUI_CB)(IN BYTE_T *oui, IN BYTE_T oui_len);
// 示例
OPERATE_RET ty_pegasus_set_oui_cb(IN BYTE_T *oui, IN BYTE_T oui_len)
```

**函数描述**

设置涂鸦oui。

**参数**

| 输入/输出 | 参数名  | 描述    |
| --------- | ------- | ------- |
| [IN]      | oui     | 涂鸦oui |
| [IN]      | oui_len | oui长度 |

**返回值**

0 : 成功； 其他：失败错误码

**说明**

该接口内容由sdk使用者实现。

路由器无感配网只处理带有涂鸦oui的probe帧，涂鸦oui为0x68,0x57,0x2d。



##### 注册回调send_frame_cb

```c
typedef OPERATE_RET (*TY_PEGASUS_SEND_FRAME_CB)(IN CONST TY_FRAME_TYPE_E type, IN CONST UINT8_T *vsie, IN CONST UINT_T vsie_len, IN NW_MAC_S *srcmac, IN NW_MAC_S *dstmac);
// 示例
OPERATE_RET ty_pegasus_send_frame_cb(IN CONST TY_FRAME_TYPE_E type, IN CONST UINT8_T *vsie, 
                                     IN CONST UINT_T vsie_len, IN NW_MAC_S *srcmac, IN NW_MAC_S *dstmac)
    
typedef enum {
    TY_FRAME_TP_BEACON = 1,          
    TY_FRAME_TP_PROBE_REQ = 2,          
    TY_FRAME_TP_PROBE_RESP = 3,          
    TY_FRAME_TP_INVALD
        
} TY_FRAME_TYPE_E;
```

**函数描述**

发送Probe Respone帧、beacon帧。

**参数**

| 输入/输出 | 参数名   | 描述              |
| --------- | -------- | ----------------- |
| [IN]      | type     | 帧类型            |
| [IN]      | vsie     | vsie内存地址      |
| [IN]      | vsie_len | vsie长度          |
| [IN]      | srcmac   | 路由器mac地址     |
| [IN]      | dstmac   | 待配网设备mac地址 |

**返回值**

0 : 成功； 其他：失败错误码

**说明**

该接口内容由sdk使用者实现。

路由器在发送Probe Respone帧、beacon帧时，sdk使用者只需将数据分别进行转发和广播，无需关心具体数据格式。



##### 注册回调event_cb

```c
typedef VOID (*TY_PEGASUS_EVENT_CB)(IN CONST TY_PEGASUS_EVENT_E event);
// 示例
VOID ty_pegasus_event_cb(IN CONST TY_PEGASUS_EVENT_E event)
    
typedef enum {
    TY_PEGASUS_START = 1,          
    TY_PEGASUS_STOP = 2,
    TY_PEGASUS_GET_PROBE_START = 3,
    TY_PEGASUS_EVENT_INVALD
        
} TY_PEGASUS_EVENT_E;
```

**函数描述**

无感配网相关事件上报，比如上报待配网设备的Probe Request。

**参数**

| 输入/输出 | 参数名 | 描述                                                         |
| --------- | ------ | ------------------------------------------------------------ |
| [IN]      | event  | 无感配网状态<br />TY_PEGASUS_START，开启无感配网<br />TY_PEGASUS_STOP，停止无感配网<br />TY_PEGASUS_GET_PROBE_START，上报probe帧<br />TY_PEGASUS_EVENT_INVALD无效事件 |

**返回值**

无

**说明**

该接口内容由sdk使用者实现。

在event为TY_PEGASUS_GET_PROBE_START状态时，上报待配网设备发送的带有涂鸦oui的probe request帧，在上报时，应注意以下几点：

（1）待配网设备会每间隔250ms（每次连续发送5帧）发送配网请求，5s超时

（2）路由器接收到的待配网设备发送的数据帧存在内容相同的情况，相同帧没必要上报，可以过滤掉，建议使用crc等校验方式，记录前后帧crc值，只要不一样，就上报

（3）路由器在处理待配网设备的数据帧的时候，还是比较耗性能的，建议上报做成异步的方式，平时处于阻塞状态，只有在TY_PEGASUS_GET_PROBE_START状态，且接收到需要上报的数据帧的时候，才去上报，数据帧的提取参照第一点，避免出现找不到而出现配网失败的情况。



#### tuya_pegasus_rept_probe

```c
OPERATE_RET tuya_pegasus_rept_probe(const UINT8_T *vsie, const UINT_T vsie_len, NW_MAC_S *smac, NW_MAC_S *dmac)
```

**函数描述**

上报待配网设备的probe request帧。

**参数**

| 输入/输出 | 参数名   | 描述              |
| --------- | -------- | ----------------- |
| [OUT]     | vsie     | vsie内存地址      |
| [OUT]     | vsie_len | vsie长度          |
| [OUT]     | srcmac   | 待配网设备mac地址 |
| [OUT]     | dstmac   | 路由器mac地址     |

**返回值**

0 : 成功； 其他：失败错误码



#### pegasus_server_second_start

```c
int  pegasus_server_second_start(uint32_t timeout_s);
```

**函数描述**

二次配网。

**参数**

| 输入/输出 | 参数名    | 描述             |
| --------- | --------- | ---------------- |
| [IN]      | timeout_s | 二次配网超时时间 |

**返回值**

0 : 成功； 其他：失败错误码

**说明**

二次配网：指设备已经激活成功，但路由器修改了密码，导致设备掉线，传统的方式设备只有重新配网，才能上线，但是二次配网允许设备不需要重新配网可以自动连回路由器（二次配网只有路由器才有这个能力，帮住其他设备配网）。



### 路由业务流程及接口

#### tuya_route_service_init

```c
OPERATE_RET tuya_route_service_init(IN CONST TUYA_ROUTE_CBS_S *p_route_cbs, 
                                    IN CONST TUYA_ROUTE_STA_CBS_S *p_sta_cbs);
typedef struct {
    TY_ROUTE_CMD_CB         route_cmd_cb;
    TY_ROUTE_QUERY_PWD_CB   route_query_cb;
    TY_ROUTE_SET_PWD_CB     route_set_cb;
    
}TUYA_ROUTE_CBS_S;

typedef struct {
    TY_ROUTE_STA_CMD_CB     sta_cmd_cb;
    
}TUYA_ROUTE_STA_CBS_S;

typedef VOID (*TY_ROUTE_CMD_CB)(IN CONST TY_ROUTE_CMD_E cmd);
typedef VOID (*TY_ROUTE_QUERY_PWD_CB)(IN CONST CHAR_T *ssid);
typedef VOID (*TY_ROUTE_SET_PWD_CB)(IN CONST CHAR_T *ssid, IN CONST CHAR_T *pwd);
typedef VOID (*TY_ROUTE_STA_CMD_CB)(IN CONST TY_ROUTE_STA_CMD_E cmd, IN CONST CHAR_T *mac, 
                                    IN CONST UINT_T value);
//示例
VOID ty_route_cmd_cb(IN CONST TY_ROUTE_CMD_E cmd);
VOID ty_route_query_pwd_cb(IN CONST CHAR_T *ssid);
VOID ty_route_set_pwd_cb(IN CONST CHAR_T *ssid, IN CONST CHAR_T *pwd);
VOID ty_route_sta_cmd_cb(IN CONST TY_ROUTE_STA_CMD_E cmd, IN CONST CHAR_T *mac, 
                                    IN CONST UINT_T value);
```

**函数描述**

路由服务初始化接口函数。

**参数**

| 输入/输出 | 参数名      | 描述                                  |
| --------- | ----------- | ------------------------------------- |
| [IN]      | p_route_cbs | 路由业务命令、查询/设置回调结构体指针 |
| [IN]      | p_sta_cbs   | sta控制命令回调结构体指针             |

**返回值**

0 : 成功； 其他：失败错误码



##### 注册回调route_cmd_cb

```c
typedef VOID (*TY_ROUTE_CMD_CB)(IN CONST TY_ROUTE_CMD_E cmd);
typedef enum {
    TY_ROUTE_CMD_GET_ONLINE_LIST = 1,          
    TY_ROUTE_CMD_GET_BLACK_LIST = 2,
    TY_ROUTE_CMD_INVALD
        
} TY_ROUTE_CMD_E;
//示例
VOID ty_route_cmd_cb(IN CONST TY_ROUTE_CMD_E cmd);
```

**函数描述**

获取设备列表。

**参数**

| 输入/输出 | 参数名 | 描述                                                         |
| --------- | ------ | ------------------------------------------------------------ |
| [IN]      | cmd    | TY_ROUTE_CMD_GET_ONLINE_LIST，获取在线设备列表<br />TY_ROUTE_CMD_GET_BLACK_LIST，获取黑名单列表<br />TY_ROUTE_CMD_INVALD，无效命令 |

**返回值**

无

**说明**

该接口内容由sdk使用者实现。



##### 注册回调route_query_cb

```c
typedef VOID (*TY_ROUTE_QUERY_PWD_CB)(IN CONST TUYA_ROUTE_PWD_E type);
typedef enum {
    TY_ROUTE_PWD_INVALID = 0,
    TY_ROUTE_PWD_2_4G = 1,
    TY_ROUTE_PWD_5G = 2,
} TUYA_ROUTE_PWD_E;

//示例
VOID ty_route_query_pwd_cb(IN CONST TUYA_ROUTE_PWD_E type);
```

**函数描述**

查询wifi密码。

**参数**

| 输入/输出 | 参数名 | 描述                                                         |
| --------- | ------ | ------------------------------------------------------------ |
| [IN]      | type   | 查询密码命令类型<br />TY_ROUTE_PWD_INVALID，无效命令<br />TY_ROUTE_PWD_2_4G，获取2.4G WIFI密码<br />TY_ROUTE_PWD_5G，获取5G WIFI密码 |

**返回值**

无

**说明**

该接口内容由sdk使用者实现。



##### 注册回调route_set_cb

```c
typedef VOID (*TY_ROUTE_SET_PWD_CB)(IN CONST TUYA_ROUTE_PWD_E type, IN CONST CHAR_T *pwd);

//示例
VOID ty_route_set_pwd_cb(IN CONST TUYA_ROUTE_PWD_E type, IN CONST CHAR_T *pwd);
```

**函数描述**

设置wifi密码。

**参数**

| 输入/输出 | 参数名 | 描述                                                         |
| --------- | ------ | ------------------------------------------------------------ |
| [IN]      | type   | 设置密码命令类型<br />TY_ROUTE_PWD_INVALID，无效命令<br />TY_ROUTE_PWD_2_4G，设置2.4G WIFI密码<br />TY_ROUTE_PWD_5G，设置5G WIFI密码 |
| [IN]      | pwd    | 密码                                                         |

**返回值**

0 : 成功； 其他：失败错误码

**说明**

该接口内容由sdk使用者实现。



##### 注册回调sta_cmd_cb

```c
typedef VOID (*TY_ROUTE_STA_CMD_CB)(IN CONST TY_ROUTE_STA_CMD_E cmd, IN CONST CHAR_T *mac, 
                                    IN CONST UINT_T value);
typedef enum {
    TY_STA_CMD_ALLOW_NET = 1,          
    TY_STA_CMD_SPEED_LIMIT = 2,
    TY_STA_CMD_UP_LIMIT = 3,
    TY_STA_CMD_DOWN_LIMIT = 4,
    TY_STA_CMD_GET_ALL_CONFIG = 5,
    TTY_STA_CMD_INVALD
        
} TY_ROUTE_STA_CMD_E;
//示例
VOID ty_route_sta_cmd_cb(IN CONST TY_ROUTE_STA_CMD_E cmd, IN CONST CHAR_T *mac, 
                                    IN CONST UINT_T value)；
```

**函数描述**

sta控制命令回调。

**参数**

| 输入/输出 | 参数名 | 描述                                                         |
| --------- | ------ | ------------------------------------------------------------ |
| [IN]      | cmd    | TY_STA_CMD_ALLOW_NET，是否允许上网命令<br />TY_STA_CMD_SPEED_LIMIT，是否开启限速命令<br />TY_STA_CMD_UP_LIMIT，最大上行速率<br />TY_STA_CMD_DOWN_LIMIT，最大下行速率<br />TY_STA_CMD_GET_ALL_CONFIG，以上所有配置信息，<br />TTY_STA_CMD_INVALD，无效命令 |
| [IN]      | mac    | mac地址                                                      |
| [IN]      | value  | 对应命令的执行结果                                           |

**返回值**

无

**说明**

该接口内容由sdk使用者实现。



#### tuya_route_rept_online_list

```c
OPERATE_RET tuya_route_rept_online_list(IN CONST USHORT_T dev_count, 
                                        IN CONST TUYA_ROUTE_DEV_LIST_S *p_list)
typedef struct {
    CHAR_T      dev_name[TY_ROUTE_DEV_NAME_MAX]; //设备名称
    CHAR_T      mac[TY_ROUTE_MAC_STR_LEN];		//mac地址
    
}TUYA_ROUTE_DEV_LIST_S;
```

**函数描述**

上报在线设备列表。

**参数**

| 输入/输出 | 参数名    | 描述               |
| --------- | --------- | ------------------ |
| [IN]      | dev_count | 设备数量           |
| [IN]      | p_list    | 设备信息结构体指针 |

**返回值**

0 : 成功； 其他：失败错误码



#### tuya_route_rept_pwd

```c
OPERATE_RET tuya_route_rept_pwd(IN CONST TUYA_ROUTE_PWD_E type, IN CONST CHAR_T *pwd)
```

**函数描述**

上报wifi密码。

**参数**

| 输入/输出 | 参数名 | 描述     |
| --------- | ------ | -------- |
| [IN]      | type   | wifi类型 |
| [IN]      | pwd    | 密码     |

**返回值**

0 : 成功； 其他：失败错误码

#### tuya_route_rept_sta_allow_net

```c
OPERATE_RET tuya_route_rept_sta_allow_net(IN CONST CHAR_T *mac, 
                                          IN CONST BOOL_T allow, 
                                          IN CONST UINT_T sta_dpid)
```

**函数描述**

上报指定mac地址设备是否允许上网。

**参数**

| 输入/输出 | 参数名   | 描述                                 |
| --------- | -------- | ------------------------------------ |
| [IN]      | mac      | 设备mac地址                          |
| [IN]      | allow    | 是否允许上网布尔值，1-允许，0-不允许 |
| [IN]      | sta_dpid | DP点                                 |

**返回值**

0 : 成功； 其他：失败错误码

#### tuya_route_rept_sta_limit

```c
OPERATE_RET tuya_route_rept_sta_limit(IN CONST CHAR_T *mac, 
                                     IN CONST BOOL_T limit, 
                                     IN CONST UINT_T sta_dpid)
```

**函数描述**

上报指定mac地址设备是否开启了限速。

**参数**

| 输入/输出 | 参数名   | 描述                               |
| --------- | -------- | ---------------------------------- |
| [IN]      | mac      | 设备mac地址                        |
| [IN]      | limit    | 限速布尔值，1-开启限速，0-没有限速 |
| [IN]      | sta_dpid | DP点                               |

**返回值**

0 : 成功； 其他：失败错误码



#### tuya_route_rept_sta_up_limit

```c
OPERATE_RET tuya_route_rept_sta_up_limit(IN CONST CHAR_T *mac, 
                                         IN CONST UINT_T limit, 
                                         IN CONST UINT_T sta_dpid)
```

**函数描述**

上报指定mac地址设备最大上行速率。

**参数**

| 输入/输出 | 参数名   | 描述         |
| --------- | -------- | ------------ |
| [IN]      | mac      | 设备mac地址  |
| [IN]      | limit    | 最大上行速率 |
| [IN]      | sta_dpid | DP点         |

**返回值**

0 : 成功； 其他：失败错误码



#### tuya_route_rept_sta_down_limit

```c
OPERATE_RET tuya_route_rept_sta_down_limit(IN CONST CHAR_T *mac, 
                                           IN CONST UINT_T limit, 
                                           IN CONST UINT_T sta_dpid)
```

**函数描述**

上报指定mac地址设备最大下行速率。

**参数**

| 输入/输出 | 参数名   | 描述         |
| --------- | -------- | ------------ |
| [IN]      | mac      | 设备mac地址  |
| [IN]      | limit    | 最大下行速率 |
| [IN]      | sta_dpid | DP点         |

**返回值**

0 : 成功； 其他：失败错误码



#### tuya_route_rept_sta_all

```c
OPERATE_RET tuya_route_rept_sta_all(IN CONST CHAR_T *mac, 
                                    IN CONST TUYA_ROUTE_STA_CONF_S *config, 
                                    IN CONST UINT_T sta_dpid)  
typedef struct {
    BOOL_T allow;
    BOOL_T limit; 
    UINT_T up_limit; 
    UINT_T down_limit; 

} TUYA_ROUTE_STA_CONF_S;
```

**函数描述**

上报指定mac地址设备所有sta信息，包括是否允许上网，是否开启限速，最大上行速率，最大下行速率。

**参数**

| 输入/输出 | 参数名   | 描述        |
| --------- | -------- | ----------- |
| [IN]      | mac      | 设备mac地址 |
| [IN]      | config   | sta所有信息 |
| [IN]      | sta_dpid | DP点        |

**返回值**

0 : 成功； 其他：失败错误码



#### tuya_route_sta_data_parse

```c
OPERATE_RET tuya_route_sta_data_parse(IN CONST CHAR_T *data)
```

**函数描述**

解析stat的所有信息。

**参数**

| 输入/输出 | 参数名 | 描述                |
| --------- | ------ | ------------------- |
| [IN]      | data   | 接收到的云端sta信息 |

**返回值**

0 : 成功； 其他：失败错误码



### 产测相关

产测仅针对扩展sdk（带zigbee功能，无zigbee功能无需关系该章节），在用户有涂鸦dongle的前提下，可通过sdk产测api对zigbee模组硬件通信功能进行产测，函数具体使用参照以下api和创建的线程__tuya_factory_manage。



#### tuya_gw_get_zigbee_ver

```c
OPERATE_RET tuya_gw_get_zigbee_ver(CHAR_T *p_ver, UINT_T in_len)
```

**函数描述**

获取zigbee版本号。

**参数**

| 输入/输出 | 参数名 | 描述                     |
| --------- | ------ | ------------------------ |
| [OUT]     | p_ver  | zigbee模组固件版本号     |
| [IN]      | in_len | zigbee模组固件版本号长度 |

**返回值**

0 : 成功； 其他：失败错误码



#### tuya_gw_zigbee_rf_test

```c
OPERATE_RET tuya_gw_zigbee_rf_test(UINT_T channel, UINT_T number)
```

**函数描述**

设置zigbee rf测试。

**参数**

| 输入/输出 | 参数名  | 描述               |
| --------- | ------- | ------------------ |
| [IN]      | channel | zigbee信道         |
| [IN]      | number  | 设置发送的数据帧数 |

**返回值**

0 : 成功； 其他：失败错误码



#### tuya_get_zigbee_rf_test_result

```c
USHORT_T tuya_get_zigbee_rf_test_result()
```

**函数描述**

获取zigbee rf测试结果。

**参数**

| 输入/输出 | 参数名 | 描述 |
| --------- | ------ | ---- |
|           |        |      |

**返回值**

大于0 : 表示通信成功，并获取到zigbee模组返回的数据帧数； 其他：失败错误码

**说明**

tuya_gw_zigbee_rf_test中的number表示设置发送的数据帧数，tuya_get_zigbee_rf_test_result返回值表示zigbee模组成功收到多少帧，收到帧数一致，表示周围信号干扰小，没有丢包，在有干扰的情况下，可能存在接收小于设置的情况，如果多个dongle同时测试，可能会出现接收大于设置的发送数（该情况出现可能较小）。

