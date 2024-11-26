**About**  
This code was created in R 4.3.0.  
It was used to create, tune, and validate a Random Forest classification model.  
It relies on the [ground truth](https://github.com/c-tyler/shoalsRF-repository/blob/main/ground_truth.csv) dataset.  

**Setup**  
Ground truth data was imported with read.csv() and named 'df'  
The habitat classes are as follows:  
 0 - bare substrate (sand, rock)  
 1 - kelp habitat (sparse or dense kelp beds)  
 2 - mixed macroalgae (kelps and red turf present in approximately equal proportions)  
 3 - red turf habitat (assemblages of filamentous red macroalgae species)  
In this section, the following steps are taken:
 - Necessary R packages are activated
 - The numeric habitat classes in 'df' are interpreted as factors

```
library(dplyr)
library(ggplot2)
library(randomForest)
library(raster)
library(Rmisc)
library(tidyverse)
df$Xhab.class <- as.factor(df$hab.class)
```

**Setting Monte Carlo Simulation**  
Monte Carlo Cross-Validation was implemented by training the Random Forest classifier inside a for loop.  
In each iteration of the loop, the ground truth data was split into a training set and an external validation set using a different seed for random number generation.  
Simulating a model multiple times imposes greater computational costs.  
Several different simulation counts were compared to determine when a point of diminishing returns in model accuracy and variability would be achieved. 
In this section, the following steps were taken:
 - A numeric sequence describing a range of Monte Carlo simulation counts was produced
 - A nested for loop was created that would iterate over the range of counts
 - Within the for loop, ground truth data was divided into ~70% training data and ~30% validation data
 - A Random Forest model was trained and internally validated on the larger set, then externally validated on the smaller set
 - The external validation accuracy value was summarized by simulation count and graphed to visually determine the point of diminishing returns

```
output.frame <- NULL
mc.length <- seq(10, 400, 10)
for (j in 1:length(mc.length)){
  for (i in 1:mc.length[j]){
    set.seed(i)
    df.int <- df
    df.int$randcol <- sample(100, size = nrow(df), replace = TRUE)
    df.t <- df.int[df.int$randcol > 30, ]
    df.v <- df.int[df.int$randcol <= 30, ]
    df.t[16] <- NULL
    df.v[16] <- NULL
    train.rf <- randomForest(hab.class ~ ., df.t, replace= TRUE)
    t.acc <- 1 - tail(train.rf$err.rate, n = 1)[,1]
    vf <- cbind.data.frame(df.v$hab.class, (predict(train.rf, df.v, type = "response")))
    colnames(vf) <- c("t", "p")
    v.acc <- nrow(filter(vf, t == p))/nrow(vf)
    mccv.no <- mc.length[j]
    output.data <- cbind.data.frame(i, mccv.no, t.acc, v.acc)
    output.frame <- rbind.data.frame(output.frame, output.data)
  }
}
out.sum <- summarySE(output.frame, measurevar = "v.acc", groupvar = "mccv.no")
ggplot(out.sum, aes(x=mccv.no, y=v.acc))+
    geom_errorbar(aes(ymin=v.acc-se, ymax=v.acc+se))+
    geom_line()+
    geom_point()
```

175 simulations was identified as an appropriate number.  
**Variable Selection**  
In this section, the following steps were taken:
 - The Random Forest model was run with 175 simulations
 - The mean importance of each variable (in terms of Mean Decrease in Accuracy) was calculated across the simulations
 - Mean importances were assessed to determine whether any variable was deleterious to the model

```
output.frame <- NULL
for (i in 1:175){
  set.seed(i)
  df.int <- df
  df.int$randcol <- sample(100, size = nrow(df), replace = TRUE)
  df.t <- df.int[df.int$randcol > 30, ]
  df.v <- df.int[df.int$randcol <= 30, ]
  df.t[16] <- NULL
  df.v[16] <- NULL
  train.rf <- randomForest(hab.class ~ ., df.t, importance = TRUE, replace = TRUE)
  md.acc <- data.frame(t(importance(tran.rf)[, 5]))
  colnames(md.acc) <- c("depth.mda", "slope.mda", "vrm.mda", "curve.mda", "north.mda", "east.mda",
                        "g.flat.mda", "g.ridge.mda", "g.shoulder.mda", "g.slope.mda", "g.footslope.mda", "g.valley.mda",
                        "sst.med.mda", "sst.std.mda")
  output.data <- cbind.data.frame(i, md.acc)
  output.frame <- rbind.data.frame(output.frame, output.data)
}
output.frame[1] <- NULL
colMeans(output.frame)
```

As no variables had a negative mean importance, none were removed.  
Parsimony is not a priority for random forest models, especially with this relatively small number of variables.  
**Hyperparameter Tuning: Number of Trees**  
The appropriate number of trees ('ntree') to use in the model was assessed.  
As with Monte Carlo simulations, increasing the number of trees improves model performance but increases computational cost.  
In this section, the following steps were taken:
 - A numeric sequence describing a range of ntree values was produced
 - A nested for loop was created that would iterate over the range of ntree values
 - The external validation accuracy value was summarized by ntree value and graphed to visually determine the point of diminishing returns

```
output.frame <- NULL
tree.length <- seq(50, 350, 25)
for (j in 1:length(tree.length)){
  for (i in 1:175){
    set.seed(i)
    df.int <- df
    df.int$randcol <- sample(100, size = nrow(df), replace = TRUE)
    df.t <- df.int[df.int$randcol > 30, ]
    df.v <- df.int[df.int$randcol <= 30, ]
    df.t[16] <- NULL
    df.v[16] <- NULL
    tree.no <- tree.length[j]
    train.rf <- randomForest(hab.class ~ ., df.t, ntree = tree.no, replace= TRUE)
    t.acc <- 1 - tail(train.rf$err.rate, n = 1)[,1]
    vf <- cbind.data.frame(df.v$hab.class, (predict(train.rf, df.v, type = "response")))
    colnames(vf) <- c("t", "p")
    v.acc <- nrow(filter(vf, t == p))/nrow(vf)
    output.data <- cbind.data.frame(i, tree.no, t.acc, v.acc)
    output.frame <- rbind.data.frame(output.frame, output.data)
  }
}
out.sum <- summarySE(output.frame, measurevar = "v.acc", groupvar = "tree.no")
ggplot(out.sum, aes(x=tree.no, y=v.acc))+geom_errorbar(aes(ymin=v.acc-se, ymax=v.acc+se))+geom_line()+geom_point()
```

After initial assessment, this was repeated using a shorter sequence with smaller steps, seq(200, 350, 15).  
A value of 215 trees was chosen.  
**Hyperparameter Tuning: Variable Choice at Split**  
The amount of variables randomly offered to each tree at each node split ('mtry') was assessed.  
Increasing mtry increases the chance a tree can choose a "good" variable at each split, but it also increases bias, reducing variation between trees.  
This defeats the purpose of a "random" forest.  
In this section, the following steps were taken:
 - A numeric sequence describing a range of mtry values was produced
 - A nested for loop was created that would iterate over the range of mtry values
 - The external validation accuracy value was summarized by mtry value and graphed to visually identify an inflection point

```
output.frame <- NULL
mtry.length <- seq(3, 14, 1)
for (j in 1:length(mtry.length)){
  for (i in 1:175){
    set.seed(i)
    df.int <- df
    df.int$randcol <- sample(100, size = nrow(df), replace = TRUE)
    df.t <- df.int[df.int$randcol > 30, ]
    df.v <- df.int[df.int$randcol <= 30, ]
    df.t[16] <- NULL
    df.v[16] <- NULL
    mtry.no <- mtry.length[j]
    train.rf <- randomForest(hab.class ~ ., df.t, ntree = 215, mtry = mtry.no, replace= TRUE)
    t.acc <- 1 - tail(train.rf$err.rate, n = 1)[,1]
    vf <- cbind.data.frame(df.v$hab.class, (predict(train.rf, df.v, type = "response")))
    colnames(vf) <- c("t", "p")
    v.acc <- nrow(filter(vf, t == p))/nrow(vf)
    output.data <- cbind.data.frame(i, mtry.no, t.acc, v.acc)
    output.frame <- rbind.data.frame(output.frame, output.data)
  }
}
out.sum <- summarySE(output.frame, measurevar = "v.acc", groupvar = "mtry.no")
ggplot(out.sum, aes(x=mtry.no, y=v.acc))+geom_errorbar(aes(ymin=v.acc-se, ymax=v.acc+se))+geom_line()+geom_point()
```

An mtry value of 6 was chosen.  
**Hyperparameter Tuning: Maximum Nodes and Minimum Node Size**  
These hyperparameters constrain the maximum number of nodes a tree can produce ('maxnodes') and the minimum size of child nodes ('nodesize').  
As these constraints are interrelated (e.g., fewer maximum nodes means small nodes are less likely), the variables were assessed simultaneously.  
In this section, the following steps were taken:
 - A numeric sequence describing a range of maxnodes values was produced
 - A numeric sequence describing a range of nodesize values was produced
 - A nested for loop was created that would iterate over both value ranges
 - The external validation accuracies were averaged for each pair of values and displayed in a table

```
output.frame <- NULL
mn.length <- seq(4, 20, 2)
ns.length <- seq(1, 21, 2)
for (k in 1:length(mn.length)){
  for (j in 1:length(ns.length)){
    for (i in 1:175){
      set.seed(i)
      df.int <- df
      df.int$randcol <- sample(100, size = nrow(df), replace = TRUE)
      df.t <- df.int[df.int$randcol > 30, ]
      df.v <- df.int[df.int$randcol <= 30, ]
      df.t[16] <- NULL
      df.v[16] <- NULL
      mn.no <- mn.length[k]
      ns.no <- ns.length[j]
      train.rf <- randomForest(hab.class ~ ., df.t, ntree = 215, mtry = 6, 
                                maxnodes = mn.no, nodesize = ns.no, replace= TRUE)
      t.acc <- 1 - tail(train.rf$err.rate, n = 1)[,1]
      vf <- cbind.data.frame(df.v$hab.class, (predict(train.rf, df.v, type = "response")))
      colnames(vf) <- c("t", "p")
      v.acc <- nrow(filter(vf, t == p))/nrow(vf)
      output.data <- cbind.data.frame(i, mn.no, ns.no, t.acc, v.acc)
      output.frame <- rbind.data.frame(output.frame, output.data)
    }
  }
}
stat.matrix <- matrix(nrow = length(ns.length), ncol = length(mn.length))
rownames(stat.matrix) <- ns.length
colnames(stat.matrix) <- mn.length
for (j in 1:length(mn.length)){
  for (i in 1:length(ns.length)){
    stat.aggregate <- aggregate(v.acc ~ ns.no + mn.no, data = output.frame, mean)
    stat.pull <- filter(stat.agg, ns.no == ns.length[i] & mn.no == mn.length[j])[,3]
    stat.matrix[i, j] <- round(stat.pull, digits = 3)
  }
}
```

Increased constraint from these hyperparameters reduced model performance.  
Thus, model defaults (unlimited maxnodes; nodesize = 1) were used.  
**Final Outputs**  
With Monte Carlo simulation, variables, and hyperparameters finalized, the model was run once more.  
It was set up to produce the following outputs:
 - Training Accuracy (t.acc)
 - External Validation Accuracy (v.acc)
 - Per-class precision, recall, and F1 statistics
 - Per-class and model-wide variable importances
 - Classification of Isles of Shoals habitats based on underlying rasters  
    - This takes the form of a multiband raster, where each band shows the number of times each cell was assigned a particular class.
    - It can be exported with the raster::writeRaster() command.

```
output.frame <- NULL
mda.bare <- NULL
mda.kelp <- NULL
mda.mixed <- NULL
mda.red <- NULL
classified.stack <- NULL

raster.predictions <- stack(r.depth, r.slope, r.vrm, r.curve, r.north, r.east,
                            r.g.flat, r.g.ridge, r.g.shoulder, r.g.slope, r.g.footslope, r.g.valley,
                            r.sst.med, r.sst.std)
names(raster.predictions <- c("r01.depth", "r02.slope", "r03.vrm", "r04.curve", "r05.north", "r06.east",
                              "r07.g.flat", "r08.g.ridge", "r09.g.shoulder", "r10.g.slope", "r11.g.footslope", "r12.g.valley",
                              "r13.sst.med", "r14.sst.std")

for (i in 1:175){
  set.seed(i)
  df.int <- df
  df.int$randcol <- sample(100, size = nrow(df), replace = TRUE)
  df.t <- df.int[df.int$randcol > 30, ]
  df.v <- df.int[df.int$randcol <= 30, ]
  df.t[16] <- NULL
  df.v[16] <- NULL
  train.rf <- randomForest(hab.class ~ ., df.t, ntree = 215, mtry = 6, importance = TRUE, replace = TRUE)
  md.acc <- data.frame(t(importance(tran.rf)[, 5]))
  colnames(md.acc) <- c("depth.mda", "slope.mda", "vrm.mda", "curve.mda", "north.mda", "east.mda",
                        "g.flat.mda", "g.ridge.mda", "g.shoulder.mda", "g.slope.mda", "g.footslope.mda", "g.valley.mda",
                        "sst.med.mda", "sst.std.mda")
  t.acc <- 1 - tail(train.rf$err.rate, n = 1)[,1]
  vf <- cbind.data.frame(df.v$hab.class, (predict(train.rf, df.v, type = "response")))
  colnames(vf) <- c("t", "p")
  v.acc <- nrow(filter(vf, t == p))/nrow(vf)
  precision.bare <- nrow(filter(vf, t == "0" & p == "0"))/nrow(filter(vf, p == "0"))
  recall.bare <- nrow(filter(vf, t == "0" & p == "0"))/nrow(filter(vf, t == "0"))
  f1.bare <- 2 * (predicion.bare * recall.bare)/(precision.bare + recall.bare)
  precision.kelp <- nrow(filter(vf, t == "1" & p == "1"))/nrow(filter(vf, p == "1"))
  recall.kelp <- nrow(filter(vf, t == "1" & p == "1"))/nrow(filter(vf, t == "1"))
  f1.kelp <- 2 * (predicion.kelp * recall.kelp)/(precision.kelp + recall.kelp)
  precision.mixed <- nrow(filter(vf, t == "2" & p == "2"))/nrow(filter(vf, p == "2"))
  recall.mixed <- nrow(filter(vf, t == "2" & p == "2"))/nrow(filter(vf, t == "2"))
  f1.mixed <- 2 * (predicion.mixed * recall.mixed)/(precision.mixed + recall.mixed)
  precision.red <- nrow(filter(vf, t == "3" & p == "3"))/nrow(filter(vf, p == "3"))
  recall.red <- nrow(filter(vf, t == "3" & p == "3"))/nrow(filter(vf, t == "3"))
  f1.red <- 2 * (predicion.red * recall.red)/(precision.red + recall.red)
  output.data <- cbind.data.frame(i, t.acc, v.acc, f1.bare, f1.kelp, f1.mixed, f1.red,
                                  precision.bare, precision.kelp, precision.mixed, precision.red,
                                  recall.bare, recall.kelp, recall.mixed, recall.red, md.acc)
  output.frame <- rbind.data.frame(output.frame, output.data)
  classified.image <- predicted(raster.predictions, train.rf, na.rm = TRUE, progress = "text")
  classified.stack <- stack(classified.stack, classified.image)
  imps <- importance(train.rf)
  bare.depth <- imps[1,1]
  bare.slope <- imps[2,1]
  bare.vrm <- imps[3,1]
  bare.curve <- imps[4,1]
  bare.north <- imps[5,1]
  bare.east <- imps[6,1]
  bare.g.flat <- imps[7,1]
  bare.g.ridge <- imps[8,1]
  bare.g.shoulder <- imps[9,1]
  bare.g.slope <- imps[10,1]
  bare.g.footslope <- imps[11,1]
  bare.g.valley <- imps[12,1]
  bare.sst.med <- imps[13,1]
  bare.sst.std <- imps[14,1]
  bare.i <- cbind.data.frame(bare.depth, bare.slope, bare.vrm, bare.curve, bare.north, bare.east,
                             bare.g.flat, bare.g.ridge, bare.g.shoulder, bare.g.slope, bare.g.footslope, bare.g.valley,
                             bare.sst.med, bare.sst.std)
  kelp.depth <- imps[1,2]
  kelp.slope <- imps[2,2]
  kelp.vrm <- imps[3,2]
  kelp.curve <- imps[4,2]
  kelp.north <- imps[5,2]
  kelp.east <- imps[6,2]
  kelp.g.flat <- imps[7,2]
  kelp.g.ridge <- imps[8,2]
  kelp.g.shoulder <- imps[9,2]
  kelp.g.slope <- imps[10,2]
  kelp.g.footslope <- imps[11,2]
  kelp.g.valley <- imps[12,2]
  kelp.sst.med <- imps[13,2]
  kelp.sst.std <- imps[14,2]
  kelp.i <- cbind.data.frame(kelp.depth, kelp.slope, kelp.vrm, kelp.curve, kelp.north, kelp.east,
                             kelp.g.flat, kelp.g.ridge, kelp.g.shoulder, kelp.g.slope, kelp.g.footslope, kelp.g.valley,
                             kelp.sst.med, kelp.sst.std)
  mixed.depth <- imps[1,3]
  mixed.slope <- imps[2,3]
  mixed.vrm <- imps[3,3]
  mixed.curve <- imps[4,3]
  mixed.north <- imps[5,3]
  mixed.east <- imps[6,3]
  mixed.g.flat <- imps[7,3]
  mixed.g.ridge <- imps[8,3]
  mixed.g.shoulder <- imps[9,3]
  mixed.g.slope <- imps[10,3]
  mixed.g.footslope <- imps[11,3]
  mixed.g.valley <- imps[12,3]
  mixed.sst.med <- imps[13,3]
  mixed.sst.std <- imps[14,3]
  mixed.i <- cbind.data.frame(mixed.depth, mixed.slope, mixed.vrm, mixed.curve, mixed.north, mixed.east,
                             mixed.g.flat, mixed.g.ridge, mixed.g.shoulder, mixed.g.slope, mixed.g.footslope, mixed.g.valley,
                             mixed.sst.med, mixed.sst.std)
  red.depth <- imps[1,4]
  red.slope <- imps[2,4]
  red.vrm <- imps[3,4]
  red.curve <- imps[4,4]
  red.north <- imps[5,4]
  red.east <- imps[6,4]
  red.g.flat <- imps[7,4]
  red.g.ridge <- imps[8,4]
  red.g.shoulder <- imps[9,4]
  red.g.slope <- imps[10,4]
  red.g.footslope <- imps[11,4]
  red.g.valley <- imps[12,4]
  red.sst.med <- imps[13,4]
  red.sst.std <- imps[14,4]
  red.i <- cbind.data.frame(red.depth, red.slope, red.vrm, red.curve, red.north, red.east,
                             red.g.flat, red.g.ridge, red.g.shoulder, red.g.slope, red.g.footslope, red.g.valley,
                             red.sst.med, red.sst.std)
  mda.bare <- rbind.data.frame(mda.bare, bare.i)
  mda.kelp <- rbind.data.frame(mda.kelp, kelp.i)
  mda.mixed <- rbind.data.frame(mda.mixed, mixed.i)
  mda.red <- rbind.data.frame(mda.red, red.i)
}
count.bare <- calc(classified.stack, na.rm = TRUE, fun = function(x, na.rm){sum(x == 0)})
count.kelp <- calc(classified.stack, na.rm = TRUE, fun = function(x, na.rm){sum(x == 1)})
count.mixed <- calc(classified.stack, na.rm = TRUE, fun = function(x, na.rm){sum(x == 2)})
count.red <- calc(classified.stack, na.rm = TRUE, fun = function(x, na.rm){sum(x == 3)})
count.brick <- brick(count.bare, count.kelp, count.mixed, count.red)
names(count.brick) <- c("c.bare", "c.kelp", "c.mixed", "c.red")
```
