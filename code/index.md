---
title: "Homework 5"
author: "Veronique Letourneau"
data: "2023-06-02"
format: 
  html:
    toc: true
    toc-location: left
    code-fold: true
    theme: yeti 
editor: visual
execute: 
  message: false
  warning: false
---

#### Links

GitHub Repo: https://github.com/veronique-let/ENVS-193DS_homework-05

## Introduction:

Sarracenia, a carnivorous plant commonly referred to as pitcher plant and indigenous to the North American region, exhibits distinct leaf morphologies that act as entrapments for arthropods (Glenn et al., 2012). The study of these plants is significant in terms of ecological value as they are threatened over much of their geographic range and are studied for their contributions towards wetland health (Wang et al., 2004). An investigation into the predictions around Serracenia biomass by evaluating characteristics like form and function (morphological), and biochemical properties (physiological) along with taxonomy can give insights about growth sequences and could help assist in population assessment while predicting responses under varying environmental conditions (Ellison et al., 2004). This research analysis focuses on Sarracenia's mass prediction by observing correlations between characteristic such as the number of leaves, feed level and physiological markers including chlorophyll content/photosynthetic rates vis-à-vis its biomass parameters which are hypothesized to be positively associated to answer the question: How do Sarracenia characteristics predict biomass? The analysis of biomass prediction in Sarracenia can therefore enhance conservation efforts and contribute to our understanding of wetland ecosystems.

## Methods

The data collection retrieved from the EDI Portal involved a manipulative feeding experiment to test the relationships between morphological and physiological characteristics of carnivorous plants, specifically examining how these relationships resemble those of non-carnivorous plants when nutrients are not limited (Ellison & Farnsworth, 2021). Variables such as photosynthetic rate, chlorophyll fluorescence, growth, architecture, and foliar nutrient and chlorophyll content were investigated. The dataset consisted of 120 observations and 32 variables (Ellison & Farnsworth, 2021), which we have processed using a quarto document in RStudio to select relevant columns for our analysis. Visualizing the presence of missing values in the data allowed us to eliminate N/A's for testing accuracy. Then, priliminary analysis of the data was performed using Pearson's r followed by its corresponding correlation plot and pairs plot to visualize the relationships between variables.

Upon generating the visualization plots, we constructed both a null and full model. The former serves the purpose of exhibiting an anticipated absence of variables' impact on the dataset whereas the latter outlines individual effects that each variable has on the response. As part of the analysis, we conducted assessments pertaining to the assumptions of normality as well as the homoscedasticity of residuals for each model. We then transformed our models into log models in order to meet the assumptions of constant variance and linearity, before creating three additional models (containing different predictors) and selecting the best predictor model for the response. Additional information pertaining to each step of the analysis is given in each sub section below.

#### Setting up

Loading in libraries

```{r libraries}
# should haves
library(tidyverse)
library(here)
library(janitor)
library(ggeffects)
library(performance)
library(naniar) 
library(flextable) 
library(car)
library(broom)

# would be nice to have
library(corrplot)
library(AICcmodavg)
library(GGally)
library(MuMIn)

```

Reading in the data

```{r reading-data}
plant <- read_csv(here("data", "knb-lter-hfr.109.18", "hf109-01-sarracenia.csv")) %>% 
  # make the column names cleaner
  clean_names() %>% 
  # selecting the columns of interest
  select(totmass, species, feedlevel, sla, chlorophyll, amass, num_lvs, num_phylls)
```

#### c. Visualizing missing data:

We plotted the visualization of missing data to demonstrate that the absence of variables in this dataset can influence the data. The presence of a significant amount of missing data pertaining to chlorophyll, biomass, and specific leaf area can compromise the accuracy of the results obtained from it.

```{r missing-data-visualization}
gg_miss_var(plant) +
labs(title = "Missing Data Visualization",
     caption = "Missing data for chlorophyll, biomass and specific leaf area") +
  theme(plot.title = element_text(size = 11, hjust = 0.5)) +
  theme(plot.caption = element_text(size = 7, hjust=0.5)) 
```

To avoid missing values skewing the analysis of a data, we subset the data by dropping N/A's to increase the accuracy of our analysis of available data.

```{r subset-drop-NA}
plant_subset <- plant %>% 
  drop_na(sla, chlorophyll, amass, num_phylls, num_lvs)
```

#### d. Creating a correlation plot:

We calculated Pearson's r and visually represented correlation using a correlation plot in order to determine the relationships between numerical variables in our data set. The plot demonstrates the association among diverse variables, where a value of +1 represents a constructive correlation and -1 denotes an opposite correlation (whereas 0 indicates no correlation).

```{r correlation-plot}
# calculate Pearson's r for numerical values ONLY
plant_cor <- plant_subset %>% 
# all column b/w feedlevel and num_phylls
  select(feedlevel:num_phylls) %>% 
# cor()  = correlation
  cor(method = "pearson")
# creating a correlation plot
corrplot(plant_cor, 
         #change the shape of what's in the cells 
         method = "ellipse",
         #adding variables over shape 
         addCoef.col = "black")
  
```

#### e. Creating a pairs plot:

We created a visualization of the relationships between variables by using a pairs plot to compare each variable against the others. In this figure, each graphical representation depicts distinct variables charted against each other. The linear plots that run diagonally exhibit the density correlation for individual variables, whereas the graphs presented in both the top and leftmost rows display an association between species and respective variable. In addition, above-diagonal squares depict Pearson correlations, while the below-diagonal plots compare two different variables through scatter plot analysis.

```{r pairs-plot}
plant_subset %>% 
  select(species:num_phylls) %>% 
  ggpairs()
```

#### f. Setting up models:

We fit multiple linear models (full and null models) in order to determine how species and physiological characteristics (or lack thereof) predict biomass.

```{r null-and-full-model}
# use 1 as the predictor
null <- lm(totmass ~ 1, data = plant_subset)
full <- lm(totmass ~ species + feedlevel + sla + chlorophyll + amass + num_lvs + num_phylls, data = plant_subset )
```

#### g. Visual and statistical assumption checks for the full model:

We visually assessed the normality and homoscedasticity of residuals using diagnostic plots for the full model.

```{r full-diagnostics}
# plot of our full model
par(mfrow =c(2,2))
plot(full)
```

The full model assumption plots show the lack of normality and heteroscedasticity spread of residuals. Therefore, to double check our assumptions, we tested for normality using the Shapiro-Wilk test (null hypothesis: variable of interest (i.e. the residuals) are normally distributed). As well as tested for homoscedasticity using the Breusch-Pagan test (null hypothesis: variable of interest (i.e. the residuals) have constant variance).

```{r test-checks}
# check normality and heteroscedasticity of our full model
check_normality(full)
check_heteroscedasticity(full)
```

#### h. Creating new full and null log objects and checking normality and heteroscedasticity:

Since the assumptions were not met above, we transformed our models into log models in order to meet the assumptions of constant variance and linearity. A second Shapiro-Wilk test and Breusch-Pagan test is performed to validate our transformation.

```{r}
null_log <- lm(log(totmass) ~ 1, data = plant_subset)
full_log <- lm(log(totmass) ~ species + feedlevel + sla + chlorophyll + amass + num_lvs + num_phylls, data = plant_subset)

# plot of our full log model
par(mfrow =c(2,2))
plot(full_log)

# check normality of our full log model using Shapiro-Wilk test 
check_normality(full_log)
# and heteroscedasticity using the Breusch-Pagan test
check_heteroscedasticity(full_log)
```

#### i. Model construction with visual and statistical assumption checks for three additional models:

The first additional model uses the predictor variable 'species', while the response variable is (log(totmass)). The 'species' predictor was chosen to see if the species of the plant affects the total mass of the plant as we hypothesized that different species are genetically predisposed to their height and weight which would affect their total biomass.

```{r model-2-log}
model2_log <- lm(log(totmass) ~ species, data = plant_subset)
# plotting model 2 log using species as predictor variable
par(mfrow =c(2,2))
plot(model2_log) 

# checking normality of our model2_log with Shapiro-Wilk test
check_normality(model2_log)
# and heteroscedasticity using the Breusch-Pagan test
check_heteroscedasticity(model2_log)
```

The second additional model uses the predictor variable 'chlorophyll', while the response variable is (log(totmass)). The 'chlorophyll' predictor was chosen to see if amount of chlorophyll in the plants affects the total mass of the plant as we hypothesized that the amount of chlorophyll produced by the plant influences biomass due to the associated sugar production in the photosynthetic process.

```{r model-3-log}
model3_log <- lm(log(totmass) ~ chlorophyll, data = plant_subset)
# plotting model 3 log using chlorophyll as predictor variable
par(mfrow =c(2,2))
plot(model3_log) 

# checking normality of our model3_log with Shapiro-Wilk test
check_normality(model3_log)
# and heteroscedasticity using the Breusch-Pagan test
check_heteroscedasticity(model3_log)
```

The third (last) additional model uses the predictor variables 'species', 'feedlevel', 'sla', 'chlorophyll', 'num_lvs' and 'num_phylls' , while the response variable is (log(totmass)). These predictors were chosen to see how the overall combination of these predictors affects the total biomass of the plant as we hypothesized that total biomass is influenced by multiple predictors rather than only one. Hence, as opposed to the full model, 'amass' was omitted in this model to observe the influence of its absence on the response variable.

```{r model-4-log}
model4_log <- lm(log(totmass) ~ species + feedlevel + sla + chlorophyll + num_lvs + num_phylls, data = plant_subset)
#plotting model 3 log using chlorophyll as predictor variable
par(mfrow =c(2,2))
plot(model4_log) 

# checking normality of our model4_log with Shapiro-Wilk test
check_normality(model4_log)
# and heteroscedasticity using the Breusch-Pagan test
check_heteroscedasticity(model4_log)
```

#### j. Evaluating multicollinearity:

We assessed multicollinearity by computing the generalized variance inflation factor for the full model and found that species exhibits the greatest GVIF value, indicating a substantial level of multicollinearity with other predictor variables. The inflated factors score is 1.23, which implies that due to multicollinearity, the standard error has increased by a factor of 1.23. Consequently, interpreting both individual effects of predictor variables and response variable becomes more difficult as species showcases an elevated degree of collinear association between these parameters.

```{r multicollinearity}
car::vif(full_log)
```

#### k. Comparing models:

We compared our five log models using Akaike's Information criterion (AIC) in order to observe which set of predictors best predicts the response. The least complex that best predicts the response is the predictor with the **lowest** value, which is Model 4 in this case (AIC = 132, in comparison to AIC = 134 for the Full Model).

```{r model-comparison}
# comparing all five log models (full, null, model 2, model 3 and model 4)
# MuMIn call for table of Akike's Information criterion values (AIC) values
MuMIn::AICc(full_log, null_log, model2_log, model3_log, model4_log)
# model selection table (tells me which predictor is least complex *and lower*)
MuMIn::model.sel(full_log, null_log, model2_log, model3_log, model4_log)
```

AIC Full model = 134

AIC Null model = 306

AIC Model 2 = 158

AIC Model 3 = 307

**AIC Model 4 = 132**

## Results:

We found that the fourth model (model4_log) which includes all predictor variables apart from 'amass' (mass-based light-saturated photosynthetic rate of the youngest leaf) best predicted total biomass in Sarracenia. This model was chosen due to it having the lowest AIC value across all other values, which in comparison to the full model including all of the predictors, model 4 has a difference in AIC value of 2. Because of this, we can conclude that model 4 demonstrates that there is a significant relationship between most of the response variables and the total biomass since the p-value is less than 0.05 (2.2e-16). With all of the p-values, except for species and feed level, being less than 0.05, this explains biologically that the combination of these predictors affects the total biomass of the plant more than just one. In addition to the model 4 results, we created a visualization of model predictions for biomass as the predictor variable and species as the response in order to observe how the Sarrencia species is correlated to biomass. The visualization shows that the Leucophylla species have the highest biomass, further demonstrating a correlation between the characteristics of the plants and improving our understanding of the relationship between different predictor variables and total biomass in Sarracenia.

Creating a summary table of Model 4

```{r summary-table}
summary(model4_log)
table <- tidy(model4_log, conf.int = TRUE, exponentiate = TRUE) %>% 
  # make it into a flextable
  flextable() %>% 
  # fit it to the viewer
  autofit()
table
```

#### c. Visualization of model predictions for biomass as the predictor variable

```{r}
# creating model of backtransformed `species`
model_pred <- ggpredict(model4_log, terms = "species", back.transform = TRUE) 
# ggpredict plot with species as response
plot(ggpredict(model4_log, terms = "species", back.transform = TRUE), add.data = TRUE) + 
# adding title/caption
  labs(title = "Visualization of model predictions for biomass as the predictor variable",
    caption = "Visualization of the relationship between species and biomass, showing a strong correlation between the predictor biomass and species response.") +
# caption plot size
  theme(plot.title = element_text(size = 10, hjust = 0.5)) +
  theme(plot.caption = element_text(size = 5.5, hjust=0.5)) 
```

## References

Ellison, A. M., Buckley, H. L., Miller, T. E., & Gotelli, N. J. (2004). Morphological variation in Sarracenia purpurea (Sarraceniaceae): geographic, environmental, and taxonomic correlates. American Journal of Botany, 91(11), 1930--1935. <https://doi.org/10.3732/ajb.91.11.1930>

Ellison, A. and E. Farnsworth. (2021). Effects of Prey Availability on Sarracenia Physiology at Harvard Forest 2005 ver 18. Environmental Data Initiative. <https://doi.org/10.6073/pasta/26b22d09279e62fd729ffc35f9ef0174>

Glenn, Bodri, M. S., & Heil, M. (2012). Fungal Endophyte Diversity in Sarracenia. PloS One, 7(3), e32980--. <https://doi.org/10.1371/journal.pone.0032980>

Wang, Hamrick, J. ., & Godt, M. J. . (2004). High genetic diversity in Sarracenia leucophylla (Sarraceniaceae), a carnivorous wetland herb. The Journal of Heredity, 95(3), 234--243. <https://doi.org/10.1093/jhered/esh043>

\
\
