---
layout: post
title: GWR在PM2.5分析中的文献综述
key: 20170321
tags: gwr
picture_frame: shadow
---

# 新思路

新模型：GTWR+高程

运用：宁波市PM2.5每个小时数据

范围：100km\*100km

自变量因子：

AOD：气溶胶光学厚度值

TEMP：温度

RH：相对湿度

WS：风速

HPBL：边界层高度（有些模型里没有）

LANDCOVER：地表覆盖

# 数据来源

AOD：

1. MODIS： Collection 5.1 MODIS Dark Target level 2  aerosol retrievals的产品  
   [https://ladsweb.modaps.eosdis.nasa.gov/](https://ladsweb.modaps.eosdis.nasa.gov/)

   The data field of  Optical\_Depth\_Land\_And\_ Ocean with best quality assurance \(AOD  Qualityflag = 3\)

   10km空间分辨率  -  
   &gt;  
   后面自己用克里金插值成5km的

2. 气象数据：

3. NARR：环境预测全球再分析的国家中心  
   [http://www.emc.ncep.noaa.gov/mmb/rreanl/](http://www.emc.ncep.noaa.gov/mmb/rreanl/)  
   （使用了ETA 模型）

   空间分辨率：32km

   包括：boundary layer height, relative humidity, air  temperature, and wind speed at 3-h intervals（每三个小时的数据），

   每天的数据由10-16点的三小时数据平均而得到

4. NLDAS\(Phase 2\)：  
   [http://ldas.gsfc.nasa.gov/nldas/ ](http://ldas.gsfc.nasa.gov/nldas/)  
    空间分辨率：1/8度，即13km  时间分辨率：1小时，

   各个指标值根据高程做了修正

   每天的数据由10-16点的时数据平均而得到。

   但好像只有北美的数据，且没有boundary layer height

5. 最后模型分别提供气象参数（混着来）

6. 气象插值方法：thin plate spline spatial interpolation method（到1km）

土地利用数据：

1. 获得一份地表覆盖数据：美国2001年30m的土地利用类型数据
   [http://www.epa.gov/mrlc/nlcd-2001](http://www.epa.gov/mrlc/nlcd-2001%29.)
2. 从中提取植被数据（植被为1，非植被为0）

# 文献综述

|                    文献                    |                    期刊                    |  年份  |   使用模型    |                    参数                    | 一些手段                                     |
| :--------------------------------------: | :--------------------------------------: | :--: | :-------: | :--------------------------------------: | :--------------------------------------- |
| High-Resolution Satellite Mapping of Fine Particulates Based on Geographically Weighted Regression | IEEE GEOSCIENCE AND REMOTE SENSING LETTERS | 2016 | GWR对比OLS  | 区域：京津冀实测点：52个 每小时[http://cnemc.cn/](http://cnemc.cn/)AOD:通过**SARA**模型计算Land Use：每年  以监测点为缓冲区1km范围进行计算工业区密度：人口密度：每年主干道长度：特定类型地类的密度（绿地、水地）：Meteorological：每年  13个站点，用最邻近发拷到监测点上  温度  湿度  大气压强建立1km格网，作分析 | 扯了SARA模型扯了GWR、OLS作出了所有区域预测的PM2.5图尺度做了1,3,10km的例子，做了对比讨论 |
| A Review on Predicting Ground PM2.5  Concentration  Using Satellite Aerosol Optical Depth |                atmosphere                | 2016 | 综述PM2.5反演 |             MLR、MEM、CTM、GWR              |                                          |
| A satellite-based geographically weighted regression model for regionalPM2.5estimation over the Pearl River Delta region in China |      Remote Sensing of Environment       | 2014 |    GWR    | 区域：珠江三角洲地区 2012/5-2013/9实测点：37个 每小时[http://cnemc.cn/](http://cnemc.cn/)AOD： Collection 5.1 MODIS Dark Target level 2  aerosol retrievals的产品[https://ladsweb.modaps.eosdis.nasa.gov/](https://ladsweb.modaps.eosdis.nasa.gov/)  The data field of  Optical\_Depth\_Land\_And\_ Ocean with best quality assurance \(AOD  Qualityflag = 3\)  10km空间分辨率  -&gt;后面自己用克里金插值成5km的Meteorological 26个站点中国气象数据网，插值到5kmTEMP：RH：WS：HPBL： | 改进了PM2.5的回归模型，用了对数出了张这一年的PM2.5预测图        |
| Estimating ground-level PM2.5concentrations in the southeasternU.S. using geographically weighted regression |          Environmental Research          | 2013 |    GWR    | 区域：美国东南地区（750km\*750km） 2003实测点：119个 每小时[http://cnemc.cn/](http://cnemc.cn/)AOD： 10km（MODIS）Meteorological NARR+NLDAS（Phrase2）混合：10kmTEMP：RH：WS：HPBL：Forest Cover：30m | 气象数据用了两个混合，需要做对比预测PM2.5和实测值的误差分析结果分析出全覆盖的PM2.5图 |
| High-Resolution Satellite-Derived PM2.5from Optimal Estimation andGeographically Weighted Regression over North America |    Environmental Science & Technology    | 2015 |    GWR    |   区域：北美地区实测点：187个AOD： 0.01度城市道路高程差大气组成   | AOD精度很高，有0.01度。通过转换                      |
| Spatiotemporal Pattern of PM2.5Concentrations in Mainland Chinaand Analysis of Its InfluencingFactors using GeographicallyWeighted Regression |            Scientific Reports            | 2017 |    GWR    | 区域：全中国　1998-2012 每年空间分辨率：1km（通过插值）![](file:///C:/Users/xwhsky/AppData/Local/Temp/enhtmlclip/Image%283%29.png) | 涉及考虑到的因子超级多空间分辨率高有技术流程图各个权重因子的全覆盖图权重大小分析 |
| Modeling the spatio-temporal heterogeneity in the PM10-PM2.5relationship |         Atmospheric Environment          | 2015 |   GTWR    |          区域：台湾 2005-2009实测点：73个          | 对比GWR、GTWR、OLS查看PM10和PM2.5的关系表现图的方式不错    |
| A Geographically and Temporally Weighted Regression Model for Ground-Level PM2.5 Estimation from Satellite-Derived 500 m Resolution AOD |              remote sensing              | 2016 |   GTWR    | 区域：中国中部（150\*50km） 2014/11-2015/2实测点：37个 每小时[http://106.37.208.233:20035/](http://106.37.208.233:20035/)AOD：SARA模式（**500m**分辨率 两天时间分辨率）[http://aeronet.gsfc.nasa.gov/](http://aeronet.gsfc.nasa.gov/)Meteorological：**3km**分辨率   每**小时**  用WRF模型提取（通过NCEP FNL Operational Model Global Tropospheric Analyses dataset（1度分辨率）来提取[http://rda.ucar.edu/dsszone/ds083.2/](http://rda.ucar.edu/dsszone/ds083.2/)）TEMP：RH：WS：PBLH：气象数据再插值到500m分辨率（cubic spline interpolation  algorithm） | 精度高，使用于城市范围分析预测每小时的PM2.5值比较GTWR、OLS、GWR、TWR |



