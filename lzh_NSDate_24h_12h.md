# 关于12小时/24小时显示导致返回nil的问题

## 原因:
通过比较后台返回的时间来进行判断event开始时间跟结束时间,利用NSDateFormatter dateformString来转换显示时间格式,结果返回nil.

## 原代码: 
```         
[NSDateFormatter setDateFormat:@"yyyy-MM-dd HH:mm:ss”];
```
HH: 代表为24小时
hh: 代表为12小时

## 修改后代码: 

```
[dateFormatter setLocale:[[NSLocale alloc] initWithLocaleIdentifier:@"en_US_POSIX"]];
[dateFormatter setDateFormat:@"YYYY-MM-dd HH:mm:ss"];

```
iOSSDK提供的日期时间格式化对象需要使用到NSLocale对象来控制日期和时间的显示，en_US为标准的格式往往使用的最多，iOS SDK提供的另一种格式en_US_POSIX,并且两种格式运行后的效果完全一样。

“en_US_POSIX”在时间上也是不变的（如果美国在未来某个时候改变日期格式，“en_US”将改变以反映新行为，但“en_US_POSIX”不会），以及在机器之间（“en_US_POSIX”在iPhone操作系统上的工作原理与在Mac OS X上的相同，在其他平台上也一样）。
