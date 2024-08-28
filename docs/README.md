# 城市道路识别赛车辆使用指南

> <font size=5>前言</font>

---

<font size=5>&emsp;&emsp;本手册旨在对船电实验**室内城市道路识别赛车**的使用流程与方法进行说明。如有任何需求，可以在BiliBili私信或微信联系。</font>

# 比赛规则简介

> <font size=5>官方规则介绍</font>

---

<div style="text-align:center;">  
    <img src="/img/map.png" style="display:inline-block;">  
</div>

<font size=5><center>__赛事地图（二代）__</center></font>



<font size=5>**1.参赛（机器人道具要求）**</font>

<font size=5>&emsp;&emsp;本赛项参赛队伍可使用推荐平台（下图所示）或者自制平台，严禁使用第三方现成平台参赛，若参赛队选择自制参赛平台，应符合以下参数要求，并将自制平台的详细参数及样图提交至赛项联系人，赛项联系人将按照大赛总规则的流程给与答复。\
&emsp;&emsp;（1）设备尺寸要求：长≥300mm，宽≥260mm，高≤320mm。（明显不属于车身整体框架的零件和结构，均不能计算在车身尺寸内）\
&emsp;&emsp;（2）本赛项底盘须采用四轮差速，严禁使用阿克曼底盘和麦克纳姆轮。\
&emsp;&emsp;（3）处理器：采用 Intel 或者 Jetson Nano 主控，运用深度学习算法。\
</font>

</font>

<div style="text-align:center;">  
    <img src="/img/car2.png" style="display:inline-block;">  
</div>

<font size=5><center>__车辆本体（二代）__</center></font>

<font size=5>**2.任务规则与得分标准**</font>

<font size=5>&emsp;&emsp;比赛开始时，智能车从起点线出发（**车头对齐起始线**），沿着车道线行驶，行驶途中需识别随机转向标志按照指示牌行驶、经过人行横道识别，来到限速环岛区域（需识别限速标志和限速解除标志），驶出环岛按照红绿灯指示行驶来到变道区域（任务加分项，可正常行驶）然后继续行驶，识别危险标识之后**车身完全驶过终点线完成比赛（车尾对其或越过终点线）**
</font>

<font size=5>&emsp;&emsp;本次赛项将采用任务得分制，总分数&ensp;=&ensp;任务分（100 分）&ensp;＋&ensp;报告分数（20 分）\
&emsp;&emsp;(1)&emsp;正常发车+5分。\
&emsp;&emsp;(2)&emsp;按照标志牌指示行驶+20 分（违反标识牌行驶+5 分）。\
&emsp;&emsp;(3)&emsp;人行横道正确停车且无压线情况+10 分（压线及停在其他区域+2 分，未停车不加分）。\
&emsp;&emsp;(4)&emsp;正确识别限速和解除限速*并有明显速度变化*+10 分\
&emsp;&emsp;(5)&emsp;识别红绿灯并成功停在黄色框内无压线情况+10 分（压线以及停在其他区域+2 分，未停车不加分）\
&emsp;&emsp;(6)&emsp;识别变道标志且进行变道行驶+30 分（未识别变道且正常行驶+10 分）\
&emsp;&emsp;(7)&emsp;成功识别危险标识并能正确行驶+10 分\
&emsp;&emsp;(8)&emsp;按要求到达终点+5 分\
&emsp;&emsp;(9)&emsp;无人车在行驶过程中车身垂直投影覆盖黄线（单轮压线），-5 分/次，如双轮或双轮以上压线计行驶失败，出局处理。\
&emsp;&emsp;(10)&ensp;**以上所有得分将在智能车跑完完整地图的前提下计入。**\
</font>

> <font size=5>规则解读</font>

---

<font size=5>&emsp;&emsp;1.车辆路牌中，左右转路牌为**裁判随机指定**，因此务必做到左右转均可实现。 \
&emsp;&emsp;2.车辆在红绿灯路口时，如果直线停车将无法看到绿灯，省组委会的商议结果是“看到红灯停车2秒后继续前进”，但国赛**按原规则不变**，所以我们的智能车在路口停车前向左偏转，使得红绿灯进入视野。 \
&emsp;&emsp;3.注意规则10，只有顺利完赛后之前的成绩才有效，否则依旧是0分（国家优秀奖买来的教训😭） \
</font>

# 还有几件事

<font size=5>&emsp;&emsp;车辆内置了一块标称14.8V，8000mAh，共计118.4wh的电池，无法带上飞机🛫，建议高铁🚝或另购小于100wh的标称14.8V的电池。\
&emsp;&emsp;主机里的环境已经被我们糟蹋的差不多了，如果需要，可以联系厂家寻求原装镜像。并附上作者的代码即可。\
&emsp;&emsp;电脑待机、车辆停止时电量显示低于 15.0v 时请立即充电，保证电池健康。\
&emsp;&emsp;所谓接下来的前期部署章节。在赛道不变的前提下不用更改。可能由于光线原因可以自行更改车道线模型，路牌模型不需要更改。在配置yolov5模型训练时，注意不要使用镜像、旋转操作强化训练集，防止左右转分不清楚。\
&emsp;&emsp;前期部署章节有厂家具体说明。详见[深度学习车使用手册v1.pdf](https://qi9206pf15p.feishu.cn/wiki/N9BCwhBFeiI6pmkGGJ2cpsUWnOh?from=from_copylink)\
&emsp;&emsp;在myself文件夹里有一些小工具，具体干嘛的试了就知道。\
&emsp;&emsp;电机上的编码器没有任何用处。\
</font>

<div style="text-align:center;">  
    <img src="/img/car.jpg" style="display:inline-block;">  
</div>

<font size=5><center>__作者与车辆本体（一代）__</center></font>

<font size=5><p align="right">作者：梁华鑫</p></font>

<font size=5><p align="right">2024年8月28日</p></font>

<font size=5><p align="right">版本v1.0</p></font>