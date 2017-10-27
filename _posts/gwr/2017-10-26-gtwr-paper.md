---
layout: post
title: Geographically and temporally weighted regression for modeling spatio-temporal variation in house prices
category: gtwr
---
Bo Huang , Bo Wu & Michael Barry 

<script type="text/javascript" src="http://cdn.mathjax.org/mathjax/latest/MathJax.js?config=default"></script>

## Abstract

By incorporating temporal effects into the geographically weighted regression (GWR) model, an extended GWR model, geographically and temporally weighted regression (GTWR), has been developed to deal with both spatial and temporal nonstationarity simultaneously in real estate market data. Unlike the standard GWR model, GTWR integrates both temporal and spatial information in the weighting matrices to capture spatial and temporal heterogeneity. The GTWR design embodies a local weighting scheme wherein GWR and temporally weighted regression (TWR) become special cases of GTWR. In order to test its improved performance, GTWR was compared with global ordinary least squares, TWR, and GWR in terms of goodness-of-fit and other statistical measures using a case study of residential housing sales in the city of Calgary, Canada, from 2002 to 2004. The results showed that there were substantial benefits in modeling both spatial and temporal nonstationarity simultaneously. In the test sample, the TWR, GWR, and GTWR models, respectively, reduced absolute errors by 3.5%, 31.5%, and 46.4% relative to a global ordinary least squares model. More impressively, the GTWR model demonstrated a better goodness-of-fit (0.9282) than the TWR model (0.7794) and the GWR model (0.8897). McNamara’s test supported the hypothesis that the improvements made by GTWR over the TWR and GWR models are statistically significant for the sample data. 

**Keywords:**geographically and temporally weighted regression; geographically weighted regression; spatial nonstationarity; temporal nonstationarity; housing price; Calgary 

## 1. Introduction 

Location and time are important determinants of real estate prices. Location is an important factor because of the spatial dependence between real estate prices, even though there tends to be spatial heterogeneity across prices over a large area. Prices of proximate houses tend to be similar because they share common local neighborhood factors, such as similar physical characteristics (age, size, and exterior and interior features) and similar neighborhood amenities (socioeconomic status, access to employment opportunities, shopping, public service facilities, schools, etc). Differences between houses in the same neighborhood tend to be determined by the size of the lot and the size and quality of the top structure.

A large-size lot, for example, tends to have a large house, a garage, and more bedrooms. In older neighborhoods, the date and quality of construction and the level of renovation tend to be major scriminating factors. 

Housing price observations in many studies tend to be geo-referenced to account for spatial autocorrelation and general neighborhood characteristics. The time of an observation also matters in the determination of real estate prices. From a modeling perspective, it is generally accepted that real estate prices depend not only on recent market events but also on their lagged prices. Temporal effects include, for example, market trends, inflationary factors, and differential rates of obsolescence related to the age distribution of houses. Depreciation of housing amenities might occur at different rates related to housing characteristics at the beginning of the study period, the original value of the house, the specific amenities included in the housing package, and other factors omitted in models that do not specifically account for intertemporal heterogeneity (Dombrow et al. 1997). 

Heterogeneous spatial and/or temporal effects may violate the basic assumption of statistical independence of observations which is typically required for unbiased and efficient estimation (Huang et al. 2009). Several studies have, therefore, tended to incorporate spatial or temporal characteristics in house rice equations to eliminate dependencies, or nonrandom effects, in the residuals. 

Prediction methods using regression can be grouped into two general categories: global and local regression models. Global spatial models are usually an improved form of the traditional hedonic model (Can 1992, Dubin 1992, Anselin 1998). Spatial or temporal effects are addressed by modeling the residual variance–covariance matrix directly or by inversing the residual variance–covariance matrix to eliminate dependency in the residuals. In Gelfand et al. (2004), a rich class of temporal hedonic models using a Bayesian framework is formulated by extending the different processes of the error term. An extended model of this study with spatially varying coefficient process is also developed in Gelfand et al. (2003). In fact, this model can be expanded to a spatio-temporal setting. Can and Megbolugbe (1997) constructed a distance-weighted average variable that captures both spatial and temporal information, which proves to be a significant explanatory variable in the hedonic model. Alternatively, Pace et al. (1998) introduced a filtering process, and their results greatly enhanced the accuracy of model estimations.

However, a major problem with global methods when applied to spatial or temporal data is that the processes being examined are assumed to be constant over space. For a specific model (e.g., the price of real estate), the assumption of stationarity or structural stability over time and space is generally unrealistic, as parameters tend to vary over the study area.

In order to capture the spatial variation, various localized modeling techniques have been proposed to capture spatial heterogeneity in housing markets. Eckert (1990) suggested that, based on the assumption that subsets are characterized by a lower variance, models generated for housing submarkets should yield greater explanatory power (and predictive accuracy) than those computed at the overall market level. Goodman and Thibodeau (1998) introduced the concept of hierarchical linear modeling, whereby dwelling characteristics, neighborhood characteristics, and submarkets interact to influence house prices. In a similar vein, McMillen (1996) and McMillen and McDonald (1997) introduced nonparametric local linear regression in nonmonocentric city models. Notably, Brunsdon et al. (1996), Fotheringham et al. (1996), and Fotheringham et al. (2002) proposed geographically weighted regression (GWR) as a local variation modeling technique.

GWR allows the exploration of the variation of the parameters as well as the testing of the significance of this variation, and this methodology has, therefore, received considerable attention in recent years. Pavlov (2000), Fotheringham et al. (2002), and Yu (2006) have all applied the GWR or GWR-similar methodology to housing markets. Brunsdon et al. (1999), in a study of house prices in the town of Deal in south-eastern England, examined the determinants of house price with GWR and found that the relationship between house price and size varied significantly through space. Despite the strength of GWR as opposed to
global models and the success of GWR in capturing spatial variations, applying a GWR model to house price analysis that incorporates temporal effects remains a relatively unexplored area.

The objective of this article is to extend the traditional GWR model to problems involving both spatial and temporal nonstationarity in real estate data. This study seeks to contribute to the literature on the topic in the following three ways. First, we extend the traditional GWR model with temporality into a geographically and temporally weighted regression (GTWR) model and apply it to spatio-temporal real estate data analysis. Second, we propose the use of McNamara’s test for comparing the statistically significant difference between the estimation methods according to their accuracies (Foody 2004). Third, we examine and compare the hedonic model, temporally weighted regression (TWR), GWR, and GTWR for modeling housing prices by means of a case study in the Canadian city of Calgary.

This article is structured as follows. In Section 2, we present a basic framework for GWR and then extend it to include temporal data. Section 3 offers some key technical implementation details of the GTWR model, including spatial and parameter optimal selection, and model comparison criterion. In Section 4, a case study of housing prices in Calgary is reported. In Section 5, the results for different models are compared and analyzed. Finally, we summarize and draw conclusions.

## 2. Geographically weighted regression model 

### 2.1 The model and the parameter estimation 

The GWR model extends the traditional regression framework by allowing parameters to be estimated locally so that the model can be expressed as 
$$
Y_i=\beta_0(u_i,v_i)+\sum_k\beta_k(u_i,v_i)X_{ik}+\varepsilon_i   \quad  i=1,2\cdots n \quad \quad (1)
$$
where $(u_i,v_i)$ denotes the coordinates of the point i in space, $\beta_0(u_i,v_i)$ represents the intercept value, and $\beta_k(u_i,v_i)$ is a set of values of parameters at point i. Unlike the ‘fixed’ coefficient estimates over space in the global model, this model allows the parameter estimates to vary across space and is therefore likely to apture local effects. 

To calibrate the model, it is assumed that the observed data close to point i have a greater influence in the estimation of the k(ui,vi) parameters than the data located farther from observation i. The estimation of parameters k(ui,vi) is given by Equation (2) 
$$
\hat\beta(u_i,v_i)=[X^TW(u_i,v_i)X]^{-1}X^TW(u_i,v_i) Y  \quad \quad (2)
$$
where $W(u_i,v_i)$is an $n *n$ matrix whose diagonal elements denote the geographical weighting of bservation data for observation i, and the off-diagonal elements are zero. The weight matrix is computed for each point i at which parameters are estimated.

### 2.2 Weighting matrix specification 

The weight matrix in GWR represents the different importance of each individual observation in the data set used to estimate the parameters at location i. In general, the closer an observation is to i, the greater the weight. Thus, each point estimate i has a unique weight matrix.  

In essence, there are two weighting regimes that can be used: fixed kernel and adaptive kernel. For the fixed kernel, distance is constant but the number of nearest neighbors varies. For the adaptive kernel, distance varies but the number of neighbors remains constant. The most commonly used kernels are Gaussian distance decay-based functions (Fotheringham et al. 2002): 
$$
W_{i,j}=\exp(-\frac {d_{ij}^2}{h^2})  \quad \quad (3)
$$
where $h$ is a non-negative parameter known as bandwidth, which produces a decay of influence with distance and $d_{ij}$ is the measure of distance between location $i$ and $j$. Using point coordinates$ (x_i,y_i)$ and $ (x_j,y_j)j$, the distance is usually defined as a Euclidean distance
$$
d_{ij}=\sqrt{(x_i-x_j)^2+(y_i-y_j)^2} \quad \quad (4)
$$
According to Equations (3) and (4), if i and j coincide, the weight of that observation will be unity, and the weight of other data will decrease according to the Gaussian curve when the distance between i and j  increases. Other commonly used weighting functions include the bisquare function (Fotheringham et al. 2002) and the tri-cube kernel function (McMillen 1996). 

To avoid (1) exaggerating the degree of nonstationarity present in the areas where data are sparse or (2) mask subtle spatial nonstationarity where the data are dense (Paez et al. 2002), adaptive weighting functions are used to change the kernel size to suit localized observation patterns. Kernels have larger bandwidths where the data points are sparsely distributed and smaller ones where the data are plentiful. By adapting the bandwidth, the same number of nonzero weights is used for each regression point i in the analysis. For example, the adaptive bi-square weighting function is the following: 
$$
W_{ij}=
\begin{cases}
[1-(\frac{d_{ij}}{h_i})^2]^2 &\mbox{if $d_{ij}<h_i$ }\\
0 &\mbox{otherwise}\\
\end{cases}  \quad \quad (5)
$$
where $h_i$stands for the bandwidth particular to location $i$.

### 2.3 Choosing an appropriate bandwidth 

In the process of calibrating a GWR model, the weighting model should first be decided. This can be done by cross-validation. Suppose that the predicted value of $y_i$ from GWR is denoted as a function of $h$ by ,$\hat y_i(h)$ the sum of the squared error may then be written as 
$$
CVRSS(h)=\sum _i(y_i-\hat y_{\neq1}(h))^2  \quad \quad (6)
$$
In practice, plotting CVRSS(h) against the parameter h can provide guidance on selecting an appropriate value of the parameter or it can be obtained automatically with an optimization technique by minimizing Equation (6) in terms of goodness-of-fit statistics or the corrected Akaike information criterion (AIC) (Hurvich et al. 1998, Fotheringham et al. 2002). 

## 3. Extending GWR with temporal variations 

Since complex temporal effects can also lead to nonstationarity in real estate prices, this article demonstrates how to incorporate temporal information in the GWR model to develop a GTWR model that captures both spatial and temporal heterogeneity and improves its goodness-of-fit. 