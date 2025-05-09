

## 1. Pynetbox提取设备列表

```py
#Pynetbox提取设备列表
import pynetbox
import requests


def get_device_ips():
    # 创建自定义 session 处理 SSL 验证
    session = requests.Session()
    session.verify = False  # 禁用 SSL 验证（仅测试环境使用）

    # 正确初始化 pynetbox
    nb = pynetbox.api(
        url='https://netdb.fujiang.com/',  # 仅保留根域名
        token='19cab467bc8e8',  # 替换实际 token
        threading=True
    )
    nb.http_session = session  # 注入自定义 session

    # 构建过滤参数
    filters = {
        'status': 'active',
        'manufacturer_id': 12,
        'cf_是否为核心服务': ''  # 中文参数需确保编码正确
    }

    try:
        # 获取设备列表（显式设置分页）
        devices = nb.dcim.devices.filter(**filters, limit=50)

        # 提取数据并处理异常
        return {
            device.name: device.primary_ip.address if device.primary_ip else None
            for device in devices
            if hasattr(device, 'primary_ip')  # 防止属性缺失
        }
    except Exception as e:
        print(f"Error: {str(e)}")
        return {}


# 使用示例
if __name__ == "__main__":
    device_ip_mapping = get_device_ips()
    print(f"Found {len(device_ip_mapping)} devices:")
    for name, ip in device_ip_mapping.items():
        print(f"{name}: {ip}")

```

```bash
BJ-HD-B1F-R1-SFW-01a: 172.18.125.132/25
BJ-HD-B1F-R1-SFW-01b: 172.18.125.133/25
BJ-JXQ-D5-CiFW01-H07U12: 192.168.43.18/24
BJ-JXQ-D5-CiFW02-H08U12: 192.168.43.19/24
BJ-JXQ-D5-IFW01-H07U6: 192.168.43.22/24
BJ-JXQ-D5-IFW02-H08U6: 192.168.43.23/24
BJ-JXQ-D5-LSFW01-H07U08: 192.168.43.20/24
BJ-JXQ-D5-LSFW01-H08U08: 192.168.43.21/24
```

---

## 2.Pysnmp测试和分类

```py
#环境安装
 pip3 install pynetbox
 pip install pysnmp==4.4.12 pyasn1==0.4.8
```

```py
import pynetbox
import requests
from pysnmp.hlapi import *
import urllib3

# 禁用SSL警告
urllib3.disable_warnings()

# 配置参数
SNMP_COMMUNITY = 'Y(K)D@6N'
SNMP_OIDS = [
    ('1.3.6.1.4.1.28557.2.33.1.1.1.1.2.1',),
    ('1.3.6.1.4.1.28557.2.33.1.1.1.1.2.2',)
]


def get_device_ips():
    """从NetBox获取设备信息"""
    session = requests.Session()
    session.verify = False

    nb = pynetbox.api(
        url='https://netdb.fujiang.com/',
        token='19caa4c1b467bc8e8',
        threading=True
    )
    nb.http_session = session

    devices = nb.dcim.devices.filter(
        status='active',
        manufacturer_id=12,
        role_id=62,
        cf_是否为核心服务='',
        limit=50
    )

    return [
        {
            "name": device.name,
            "ip": device.primary_ip.address.split('/')[0] if device.primary_ip else None
        }
        for device in devices
        if device.primary_ip
    ]


def snmp_get(ip):
    """执行SNMP GET请求（最终修正版）"""
    last_error = None

    for oid in SNMP_OIDS:
        try:
            error_indication, error_status, error_index, var_binds = next(
                getCmd(SnmpEngine(),
                       CommunityData(SNMP_COMMUNITY),
                       UdpTransportTarget((ip, 161), timeout=3, retries=1),
                       ContextData(),
                       ObjectType(ObjectIdentity(*oid)))
            )

            # 成功获取到值
            if not error_indication and not error_status:
                result = var_binds[0][1].prettyPrint()
                # 检查结果是否为指定的No Such Object或No Such Instance
                if "No Such Object currently exists at this OID" in result or "No Such Instance currently exists at this OID" in result:
                    return None, "不支持该OID"
                return result, None

            # 记录错误类型
            if error_status:
                error_type = error_status.prettyPrint()
                if error_type in ['noSuchInstance', 'noSuchObject']:
                    return None, "不支持该OID"
                else:
                    last_error = f"SNMP协议错误: {error_type}"
            elif error_indication:
                if 'No SNMP response' in str(error_indication):
                    last_error = "SNMP连接失败"
                else:
                    last_error = f"SNMP错误: {error_indication}"

        except Exception as e:
            last_error = f"SNMP连接失败: {str(e)}"

    return None, last_error if last_error else "未知错误"


def main():
    devices = get_device_ips()

    results = {
        "success": [],
        "unsupported": [],
        "failed": []
    }

    for device in devices:
        name = device['name']
        ip = device['ip'] or "N/A"

        if not device['ip']:
            results["failed"].append((name, ip, "无IP地址"))
            continue

        result, error = snmp_get(device['ip'])
        if result is not None:
            results["success"].append((name, ip, result))
        elif error == "不支持该OID":
            results["unsupported"].append((name, ip, error))
        else:
            results["failed"].append((name, ip, error))

    # 打印统计信息
    print(f"\n{' 统计结果 ':=^60}")
    print(f"成功设备: {len(results['success'])} 台")
    print(f"不支持设备: {len(results['unsupported'])} 台")
    print(f"失败设备: {len(results['failed'])} 台")

    # 打印详细结果
    def print_section(title, data):
        print(f"\n{' ' + title + ' ':-^60}")
        for item in data:
            status = item[2] if title in ["失败设备", "不支持设备"] else ""
            print(f"{item[0]:<25} {item[1]:<15} {item[2] if title == '成功设备' else status}")

    print_section("成功设备", results["success"])
    print_section("不支持设备", results["unsupported"])
    print_section("失败设备", results["failed"])


if __name__ == "__main__":
    main()
```

```md
=========================== 统计结果 ===========================
成功设备: 71 台
不支持设备: 46 台
失败设备: 4 台
................................................
--------------------------- 成功设备 ---------------------------
BJ-JXQ-D5-IFW01-H07U6     192.168.43.22   552576
BJ-JXQ-D5-IFW02-H08U6     192.168.43.23   552576
................................................
-------------------------- 不支持设备 ---------------------------
CQ-GY-8F-IFW01            172.31.74.130   不支持该OID
CQ-QLZM-5F-IFW01          11.41.193.15    不支持该OID
................................................
--------------------------- 失败设备 ---------------------------
IT-TEST-IPV4-IFW01        11.37.1.7       SNMP连接失败
IT-TEST-IPV4-IFW02        11.37.1.8       SNMP连接失败
IT-TEST-IPV6-IFW01        11.37.1.9       SNMP连接失败
IT-TEST-IPV6-IFW02        11.37.1.10      SNMP连接失败

进程已结束，退出代码为 0
```



