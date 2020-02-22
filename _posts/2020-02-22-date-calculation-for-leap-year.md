---
layout: post
title: 闰年那些事
category: 其他
---

<a name="l68nJ"></a>
## 什么是闰年
闰年是比普通年分多出一段时间的年分，在各种历法中都有出现，目的是为了弥补人为规定的纪年与地球公转产生的差异，与闰年对应的年份称为平年。比如刚刚过去的 2019 年就是平年，而现在的 2020 年就是闰年。在大家的印象中，闰年是每 4 年一次，但是其实不是这样的，比如 1700，1800，1900 年都不是闰年，而 2000 年就是闰年。<br />闰年的计算方法：

1. 如果年份是 4 的倍数，且不是 100 的倍数，则它是闰年，比如 2020；
1. 如果年份是 400 的倍数，则它是闰年，比如 2000；
1. 其他的年份都是平年，比如 2019，1900。
<a name="hxRQu"></a>
## 闰年计算错误会导致什么问题
闰年比平年多一天，在计算机中使用时一定要注意它最主要的两个特征：

- 闰年的二月有 29 天；
- 闰年一共有 366 天。

比如为 2 月分配一个天数的数组，那么需要知道这个二月是否是闰年的二月，比如为一年的天数分配一个数组，需要确定这一年是否是闰年，否则一不小心就会造成数组越界的异常。<br />闰年计算错误比较著名的影响就是微软 Azure 2012 年的故障，根因是 Azure 会为虚拟机生成一个有效期一年的证书，下发到下面的虚拟机，一年时间的计算方式是直接在年份值上加 1，比如在 2019 年 11 月 11 日生成一个为期一年的证书，那么证书的有效期会到 2020 年 11 月 11 日，而 2020 年是闰年，二月有 29 天，在 2012 年 2 月 29 日，Azure 生成的证书有效期截止时间是 2013 年 2 月 29 日，显然这不是一个有效的日志，所以证书生成错误，导致了 Azure 的故障，故障详见文末 Reference 的第 2 篇。
<a name="dNwYR"></a>
### 常见润年计算错误

1. 闰年计算方法出错，误将能整除 4 的年份当做是闰年。
1. 在日期值上直接加减年月日。

比如 Java 的 `Calendar.set(Calendar.YEAR, year + 1)`，微软 Azure 2012 年的润年故障原因就是这样。直接加减年月日的值需要考虑最后计算出来的时间是否符合预期，尽量使用内置的方法来加减年月，比如 Java 里的 `java.util.Calendar#add` 方法。

2. 月份或年的天数使用固定长度的数组

比如对于一年的天数定义长度为 365 的数组，然后再用某天在一年中的多少天去访问数组元素，即
```java
int[] array = new int[365];
Calendar calendar = Calendar.getInstance();
calendar.set(Calendar.YEAR, 2020);
calendar.set(Calendar.MONTH, 11);
calendar.set(Calendar.DATE, 31);
int dayOfYear = calendar.get(Calendar.DAY_OF_YEAR); // 366
int val = array[calendar.get(dayOfYear)]; // ArrayIndexOutOfBoundsException
```

<a name="rbEyX"></a>
## 总结
闰年对大多数人来说，只是意味着会多上一天班，但是对于我们写的代码来说，处理错误会可能会导致故障，特别是云计算，电商，金融等行业。在平常写日期处理相关的代码时，一定要关注生成的时间是否是一个有效的日期，从细节中发现问题，避免问题。

<a name="p12RD"></a>
## Reference

1. [https://azure.microsoft.com/en-us/blog/is-your-code-ready-for-the-leap-year/](https://azure.microsoft.com/en-us/blog/is-your-code-ready-for-the-leap-year/)
1. [https://azure.microsoft.com/en-us/blog/summary-of-windows-azure-service-disruption-on-feb-29th-2012/](https://azure.microsoft.com/en-us/blog/summary-of-windows-azure-service-disruption-on-feb-29th-2012/)
1. [https://zh.wikipedia.org/wiki/%E9%97%B0%E5%B9%B4](https://zh.wikipedia.org/wiki/%E9%97%B0%E5%B9%B4)
