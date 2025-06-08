---
title: Easy UI DataGrid 首次进入页面时，不加载任何数据
date: 2016-09-01 14:31:24
categories:
 - 前端
tags:
 - Javascript
 - Easy UI
description: 提供了一种Easy UI的DataGrid 组件第一次不加载数据的解决方案
permalink: /posts/2.html
---
# Easy UI DataGrid 首次进入页面时，不加载任何数据
首次不加载数据问题，必须搞明白如何才能不加载数据。根据Easu UI的官方API： http://www.jeasyui.com/documentation/

仔细观察DataGrid的事件当中有一个这样的描述：

![Easy UI table的官方说明](/images/easyui.png)

根据这个我们给onBeforeLoad事件绑定如下的事件:

```javascript
onBeforeLoad: function (param) {
                var firstLoad = $(this).attr("firstLoad");
                if (firstLoad == "false" || typeof (firstLoad) == "undefined")
                {
                    $(this).attr("firstLoad","true");
                    return false;
                }
                return true;
            }
```

这段代码的主要意思就是:去查找当前datagrid的属性firstLoad，如果是false或者没有，那么返回false。同时设置firstLoad属性为true，否则的话（认为是true）就返回true.
很显然第一次加载的时候默认肯定是没有这个属性的，那就会返回false。根据API我们已经知道，如果该事件返回false，datagrid就不会加载数据，从而在实现第一次不加载数据。而第二次的的时候firstLoad已经被设置为true，所以它会返回true，datagrid就会加载数据. 同时如果手动给table加上了firstLoad属性为true，那么datagrid也还是会在第一次加载时就load数据。
