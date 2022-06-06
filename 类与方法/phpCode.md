#### ip地址验证
- 正则验证
```
//仅支持IP4
function isValidIpAddress(string $ipAddress)
{
    //分析IP地址的组成特点：250-255、200-249、0-199
    //0-199：这个可以继续分拆:
    //100-199：三位数，百位是1，十位是09，个位是09，用正则表达式可以写成：1\d{2}
    //10-99： 二位数，十位是19，个位是09，用正则表达式可以写成：[1-9]\d
    //1-9： 一位数，个位是0~9，用正则表达式可以写成：\d

    $regex = '/^(1\d{2}|2[0-4]\d|25[0-5]|[1-9]\d|[1-9])\.(1\d{2}|2[0-4]\d|25[0-5]|[1-9]\d|\d)\.(1\d{2}|2[0-4]\d|25[0-5]|[1-9]\d|\d)\.(1\d{2}|2[0-4]\d|25[0-5]|[1-9]\d|\d)$/';
    return preg_match($regex, $ipAddress);
}
```
- 现成工具函数实现
```
//仅支持IP4，可通过参数验证IP4和IP6
function isValidIpAddress(string $ipAddress)
{
    return filter_var($ipAddress, FILTER_VALIDATE_IP, FILTER_FLAG_IPV4);
}
```

- 不使用工具函数，自行验证

```
https://leetcode-cn.com/problems/validate-ip-address/
```

