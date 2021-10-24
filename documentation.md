## Introduction:
In internal combustion engines, an accurate estimate of the air entering the engine is very important to determine the amount of fuel to be injected across driving conditions based on performance and emission requirements. The driver controls the throttle position and this along with atmospheric conditions and the current engine state leads to a certain amount of air entering the system. 
write more??

## Aim of this project:
1. Exploratory analysis for finalising the best interpretable and analytic model valid under steady state. Done.
2. Devising an optimal design process for interpreting model parameters for a new system to reduce cost of calibration activity necessary. Ongoing.

## Future possibilities:
1. Exploring models which offer better prediction accuracy while sacrificing interpretability. ex. tree based models, neural networks.

**Vehicle used:**

TVS Jupiter with a 125cc engine. The vehicle currently does not have provision for secondary air injection/exhaust gas recirculation. For idling air actuation, the vehicle has a stepper motor for fine control.

### List of inputs being measured in the system that affect air flow:
1. Ambient pressure (uncontrolled, measured)
2. Ambient temperature (uncontrolled, measured)
3. Throttle angle/ TPS position (controlled, measured)
4. Manifold absolute pressure/MAP (uncontrolled, measured)
5. Manifold air temperature/MAT (controlled, measured)
6. Cylinder head temperature/CHT (controlled, measured) 
7. Engine crank speed/RPM (uncontrolled, measured)
8. Rate of change in TPS position (controlled, measured)

* 7- Input additionally necessary for transient analysis*


### Description of the data collection apparatus:
The vehicle was mounted on a chassis dynamometer to offer variable load and thus get different vehicle speeds corresponding to the same throttle position. The torque measurement can help measure the resistance offered by the dynamometer to the rotation of the wheel. Since we wish to understand the air flow characteristics to achieve complete combustion, it is better to lock the transmission to a fixed gear ratio so that the wheel and piston move together. However, care needs to be taken so that resistance torque doesn't wear out the rubber liner of the clutch.
Using CANape, the variables 1, 2, 3, 5, 6 are measured and picoscope was used to store the entire trace for 4. The entire trace was stored to consider test orifice flow equations in the future without having to collect new data.
Syncing done between CANape 19 and picoscope files by looking vehicle starts or throttle increases. This is sufficient for now because most of CANape variables are quasistatic. 

### Assumptions under which the analysis is made:
1. Transport delay-
There isn't reliable information about the transport delay from the addition of air charge into the cylinder to getting the wideband voltage which is dependent on the partial pressure of oxygen in the exhaust gas post combustion. This is something which needs to be understood better before transient analysis can be done. As a consequence, all of the analysis is done at steady state using cross-sectional data (data at a particular point in time)

**Target variable:**

Measured air per thermocycle backcalculated using the controller injection dwell input along with AFR ratio provided by the wideband sensor. 

### Stock strategies for steady state:
Speed density method is the strategy currently employed on the vehicle for steady state air mass estimation. Volumetric efficiency (VE) tries to quantify the filling efficiency of the combustion engine across various operating condtions. Volumetric efficiency is defined as the ratio of volume of actual air in the cylinder at BDC (before the power stroke) divided by the displacement volume of the cylinder. This can be written using air flow rate (measured in systems using mass airflow sensor), and the density of air occupying the cylinder. In our setup, instead of a mass airflow sensor, mass of actual air can be written using mass of fuel added every thermocycle and the corresponding air:fuel ratio post combustion as measured by the wideband sensor. _(This assumes that we have an average value for the transport delay but it is taken to be 0 in this analysis and this needs to be explored more in the future)_
To write the density of air, ideal gas equation is used which requires the pressure and the temperature of the air. If we wish to write the density of the air in the cylinder at BDC before the power stroke, we need the cylinder pressure and air charge temperature value at BDC. However, in most real life applications, the density of air is written using manifold air pressure and cylinder head temperature. This could be because those are the best proxies available in the system. _(A cost-benefit analysis favours the current sensors because current sensors along with a thorough calibration of primary tables and correction tables in various regimes help meet the BSVI emission requirements. ex. Cylinder pressure sensors are much more expensive than manifold pressure sensors so they are avoided.)_

air mass actual per thermocycle = fuel mass per thermocycle x instantaneous air:fuel ratio

air mass actual = MAP &times; (M<sub>air</sub>/R<sub>gas</sub>) &times; CHT &times; VE

_M<sub>air</sub> = molecular mass of air_;

_R<sub>gas</sub> = universal gas constant_

Volumetric efficiency can be interpreted as the link between real world and the theoretical model. It is a value between 0 and 1 and is closer to 0.9 at wide open throttle conditons when the pressure head loss across the orifice is lowest. _(more obstruction = greater pressure head loss = lower fluid flow velocity = less efficient filling)_ It is generally calibrated against two of the possible three inputs like MAP, CHT and engine speed for example VE(MAP, speed). This is the understanding expressed about the speed density method in literature to my understanding. However, MAP at which point in time is the best for density estimation wasn't clearly specified in any of the papers referred during this project.

Three candidates for the value of MAP have been considered in this analysis. The three candidates are:
1. PreMAP - MAP value before the start of the suction stroke. 
2. PostMAP - MAP value at the end of the suction stroke. Presumably, the intake valve closes after this as the compression stroke begins.
3. SMAP - average MAP value during the suction stroke. Assuming that the suction stroke lasts 180 degrees and the crank case has 24-2 teeth configuration, the average is calculated using MAP values at the 12 teeth in the suction stroke. In case of a missing tooth, either the MAP value before or after the missing tooth is used.
 
If the physical interpretation for VE expressed above is correct, PostMAP is a good candidate if the pressure has equalized between the intake manifold and the cylinder by the end of the suction stroke. However it needs to be checked whether this pressure equalization happens at all conditons and whether higher engine speeds are just too fast for this to occur in every thermodynamic cycle.

Ultimately, speed density method is a calibration technique. A theoretical air mass estimate _MAP &times; M<sub>air</sub> &divide; (R &times; CHT)_ is contructed using sensor data and finally multiplied by VE using a lookup table based on past calibration. If the current operating condition's MAP and speed value falls between past values, the VE value used is the one interpolated over nearby values. This speed density model as implemented on the controller currently serves as the benchmark for all following models.

**Statistical interpretation of the speed-density method as implemented on the controller using interpolation:**

The speed density method on the controller is a form of double classification model (a model with discrete target values). This statement will help understand the working of the model. 
/the speed density m the classification can be seen as a form of nearest neighbours clustering.
 
If the actual air mass per thermocycle is on the Y-axis and the theoretical air mass estimate per thermocycle is on the X-axis, volumetric efficiency is the slope of the line passing through the origin in this graph. This is our first graph called graph-A. The model attempts to group operating conditions which have a similar VE by identifying the slope of the line to which every operating condition is closest to. Let's call this 'type-A similarity'. This is the first step in the model and can be called as a form of classification because the chosen values for slopes are discrete and finite. Note that the classification boundaries in graph A are straight lines passing thorugh the origin. 
The next step in the model is finding a way to reliably predict type-A similarity using the two inputs against which the MAP should be calibrated against. Known candidates for the inputs of the 2D map for volumetric efficiency are MAP, TPS and engine speed. Consider the example where MAP and engine speed are the inputs for calibration. Plotting the engine speed on Y-axis and MAP value on the X-axis of the operating conditions, graph-B is obtained. Ideally, type-A similar operating condtions should lie 'close enough' in graph-B so that they have similar values for MAP and engine speed called 'type-B similarity'. Since, the calibration is done using a 2D table with equidistant values of MAP and engine speed, it is equivalent to describing the classification boundaries between opperating conditions as gridlines imposed on graph-B. Type-A similar conditions should lie in the same square block in graph-B so that the calibration is effective.   

The purpose of the calibration activity is finding parameters for the classification boundaries for both similarities defined in the system. Thus, we could choose a different type of pressure for each of the two graphs. For example, using SMAP as the value of MAP in graph A to define type A similarity and using PostMAP as the value of MAP in graph B. 

**Obervations on graph-A and graph-B between PreMAP, PostMAP and SMAP:**

The SMAP and PostMAP have an advantage over PreMAP because they have a greater span between their minimum and maximum values. This is also reflective of the fact that PreMAP just helps us understand whether the intake manifold equalized back to atmospheric pressure before the intake valve opened or not. A greater span helps in defining classification boundaries which are sufficiently distant from each other and thus lead to lesser false cases. This is necessary to make robust choices in the presence of noise (percent based noise versus absolute values of noise). This effectively rules out PreMAP as a candidate for both the first and second graphs unless we consider normalizing the PreMAP values using min-max of PreMAP values. However, this will be a strictly mathematical device which atleast makes sense in the graph-B for calibration but not at all in graph-A where the ideal gas equation is used to describe the density of air charge. Using PreMAP results in calculating the density of air in the intake manifold before suction stroke starts which has no physical interpretation. 

Comparing between SMAP and PostMAP in graph-A, the slope in some cases is greater than 1 in graph-A using PostMAP. This will imply a volumetric efficiency greater than 1 and thus violating the interpretation of volumetric efficiency as volume fraction of combustion cylinder filled as discussed in literature. For graph-A, SMAP is perhaps the best candidate for MAP value.  

Comparing between SMAP and PostMAP in graph-B, SMAP and PostMAP provide good clustering largely similar to type-A however this is tested properly by training a regression model in the next section.

//
to understand which variable contains the most information, we could calculate the r square value of this pressure as the dependent variable and other variables
on the right as independent variables. however, this is no understanding on how the independent variables need to be combined to form terms on the left. 

but just based on how it is constructed, we know that smap contains more information than premap or post map because they both contribute to the average which is the smap 
value. in that sense, is it always going to contain more information? this is worth thinking about.

all of this is great but we can just look at the two variable regression on the entire dataset and what we observe is that the rsquare value is around 0.85.
we can thus look at other models to see if they do a better job of predicting this curve.
//

### Regression modelling:

A regression model proceeds by assuming that the true relationship between the dependent variable and independent variables is given by 
y = f(x<sub>i</sub>) + &epsilon; where &epsilon; stands for statistical error due to sampling. However, this relationship needs to be estimated using sampled data and thus fitting a model leads to y = f<sup>^</sup>(x<sub>i</sub>) + &epsilon;<sup>^</sup>. Here &epsilon;<sup>^</sup> is called the residual term which stands for the sum of any missing variables or functional forms in the assumed relationship and the aforemoentioned statistical error due to sampling. The goal is to find a model which accounts for most of the variation in y and subsequently &epsilon;<sup>^</sup> is due to sampling error. However, if the model isn't a good estimate of the true relationship, the errors might show a trend and could be correlated with the independent variables and/or themselves. 

Under the Gauss-Markov assumptions, ordinary least squares regression leads to the best linear unbiased estimates (BLUE) for the parameters of a linear regression model

z = &alpha; &times; x + &beta; &times; y + &epsilon;

The Gauss-Markov assumptions are:
1. Linearity in parameters
In linear regression, the target variable is a linear combination of the parameters but not necessarily of the independent variables. For example, z = &alpha; &times; x + &beta; &times; y + &gamma; &times; x<sup>2</sup> + &epsilon; 
Here, the greek alphabets stand for parameters, the english alphabets stand for dependent and independent variables and &epsilon; for the irreducible normally distributed error. The parameters are found by fitting the model to the data using different fitting techniques. 

2. Exogeneity or zero conditional mean of errors
This means that the independent variables are assumed to be free of measurement error. Also, the mean of the residual terms for any given measurement variable is 0. This could be violated due to an omitted variable or any missing functional form/interaction term.

3. Linear independence or no perfect multicollinearity
All the independent variables should be linearly independent. In order for the inverse of the design matrix to be defined, the design matrix should have full rank. Even if perfect multicollinearity doesn't exist, presence of some level of multicollinearity leads to larger confidence intervals for parameter estimates and subsequent problems with interpretation of the model. It is possible that the predictions of the model aren't affected provided that the model is tested in the region where the multicollinearity nature remains consistent.

4. Constant variance of error terms or homoscedasticity
If the variance of residuals changes with changes in independent variables, the assumption of identical normally distributed errors doesn't hold. This again could be due to an omitted variable or any missing functional form/interaction term.

5. No autocorrelation of errors
If the residuals correlate with lagged residuals, it signifies an interdependence between the error terms and thus the assumption of independence of errors is violated.

Writing the linear regression model corresponding to speed density method,  

air mass actual = MAP &times; (M<sub>air</sub>/R<sub>gas</sub>) &times; CHT &times; VE

VE = &alpha; &times; MAP + &beta; &times; RPM + &epsilon;

Using the two equations,
air mass actual = &alpha; &times; (M<sub>air</sub>/R<sub>gas</sub>) &times; MAP<sup>2</sup>/CHT + &beta; &times;  (M<sub>air</sub>/R<sub>gas</sub>) &times; (MAP&times;RPM/CHT) + &epsilon;
Thus the two independent variables are MAP<sup>2</sup>/CHT and MAP&times;RPM/CHT. 

The regression analysis has been done in `python` using a package called `pycaret`. This package itself depends on other popular data analysis packages like `numpy`, `seaborn`, `matplotlib`, `pandas`, etc. Please refer to pycaret documentation to get it running.  

`model_old.ipynb`


Considering the fact that data until now has been collected at similar ambient temperature and pressure values, the changes in the process due to variable 1 and 2 cannot be studied.




Gauss markov assumptions and verifying whether they are followed.
regression diagnostics.

how normaization was decided? by looking at the histogram of the independent variables. no where do they look like normal distributions. understand the central limit
theorem. why does the distribution of sample mean get closer and closer to normal distribution?

understanding the linear regression in terms of leverage and influential points. 

the different types of interactive residual plots like 
tukey anscombe
standardised residuals vs leverage
scale vs location plot
normalized qq plot



Hence, the independent variables used in the model are 3, 4, 5, 6, 7. 
replacing VE as a linear combination of speed and map, new equation is obtained. 

**Sidenote- Need for capturing wideband variations:**
n282_data2 contained the average injection dwell at a particular operating condition. In our system, an operating condition refers to constant or very little percentage variation in CHT, MAT, TPS, and engine speed. This lead to a single target variable while estimating the parameters and all of the intercyclic variations in the predictor variables are ignored especially when no comment can be made about percentage variation in independent and dependent variables. Correspondingly, a decision was made to collect intercyclic variations in injection dwell and also the instantaneous wideband sensor reading. n282_data3 and n282_data4 contain the data collected post the decision.




**CHT vs mat as a better temperature indicator**
vif of cht is lower than mat. 
a decision tree regressor which contained cht was able to bette characterise the lower temperature datasets. of course, this could be due to resolution of the mat sensor.
perhaps if we can measure mat more accurately, it will provide better estimates for air entering the engine, 

reasons to be wary of multicollinearity
how vif was used to decide which variables to consider in the regression model. should i try best subset regression?

cht, smap and tps are the final independent variables. 

Once, we decide on a model + the range of independent variables we can expect, we can attempt the design of the experiments which help us meet certain 
criteria for the process while a maximum prediction error across the entire expected operation range. this stands for G optimality and i need to understand 
this more. 

Leverage:

### Design of the experiment:

An experimental design sets which predictor variables to vary, over what range, and with what distribution of values (sampling plan).

G-optimality (avg leverage/max leverage)

**References:**

Topic

[a](http://www.lithoguru.com/scientist/statistics/course.html) link testing. 