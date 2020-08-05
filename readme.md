---
title: "Spect.Rmd"
author: "Shuce Zhang"
date: "2018/5/4"
---
This script is to analyze spectrum data from plate readers.
Should you have any questions please contact Shuce: shuce@ualberta.ca

## General Methods
  1. Right-click `test/raw.txt` to open in MS Excel, paste your data in, and save. Your data will be saved in tab-delimited formats.
  2. First time you run the script with `do_norm` set to FALSE. You can find the peak heights in the summary, which you may want to use as your normalization factors. Edit `norm_vec` accordingly, and then turn the `do_norm` to TRUE or 1.
  3. General users should focus on the parameters part. Do not mess up with the functions. If you would like to implement new tasks, write new functions for them.


## Setting parameters

You can customize this part to adapt to your experiments.

```{r}
setwd("G:/GitHub/Spectrum/test")             # working directory
#raw <- read.table("raw.txt", header = TRUE)                         # input file
init.df <- read.table("raw.txt", header = TRUE)
blank <- init.df$H7                                                  # identify the background well
wav <- init.df[,1]
raw <- init.df[,2:ncol(init.df)] - blank
raw <- data.frame(wav, raw)
raw$A7 <- NULL                                                      # Remove the background wells
raw$H7 <- NULL                                                      # Remove the background wells
colNum <- ncol(raw)
reorg <- 0                                  # 0 if replication are in the same row, 1 if same column
reorg.rep <- 3                              # number of samples


norm_wav_range <- c(350,390)                # range of peaks you want to label
                               
do_norm <- FALSE                            # 1 if you want to output normalized data

# normalization factor for each sample. 
# Measurements of each sample will be divided by these values, respectively 
norm_vec <- c(0.083, 0.200, 0.043, 0.152, 0.169, 0.192,
              0.210, 0.213, 0.216, 0.206, 0.193, 0.216,
              0.144, 0.207, 0.227, 0.196, 0.189, 0.204,
              0.253, 0.240, 0.217, 0.117, 0.194, 0.193,
              0.182, 0.191, 0.196, 0.210, 0.185, 0.196,
              0.168, 0.195, 0.086, 0.204, 0.177, 0.225, 
              0.216, 0.163, 0.521, 0.210, 0.278, 0.230,
              0.274, 0.246, 0.234, 0.232, 0.252, 0.310)
              #0.199, 0.183, 0.183, 1, 0.281, 0.291, 1)

                                            
#norm_update <- t(raw[11,2:colNum])
#rownames(norm_update) <- c()

#norm_vec <- norm_update                      # Comment if you want to use the old coefficents
```

## Useful functions

```{r functions, echo=FALSE}
##################Defining functions###########################
which.peaks <- function(x,partial=TRUE,decreasing=FALSE){
  if (decreasing){
    if (partial){
      which(diff(c(FALSE,diff(x)>0,TRUE))>0)
    }else {
      which(diff(diff(x)>0)>0)+1
    }
  }else {
    if (partial){
      which(diff(c(TRUE,diff(x)>=0,FALSE))<0)
    }else {
      which(diff(diff(x)>=0)<0)+1
    }
  }
}
summ.peaks <- function(x, partial=TRUE, decreasing=FALSE, wav = raw[,1], wavRan = norm_wav_range) {
  maInd <- which.peaks(x, partial = partial, decreasing = decreasing)
  wavLen <- wav[maInd]
  Int <- x[maInd]
  star <- rep("",length(maInd))
  star[wavLen >= wavRan[1] & wavLen <= wavRan[2]] <- "***"
  data.frame(maInd, wavLen, Int, star)
}
#############################Reorganizing data####################################
if (reorg) {
  df <- data.frame(raw[,1])
  raw[,1] <- NULL
  sel <- seq(from = 1, to = colNum - 1, by = reorg.rep)
  for (i in 1:reorg.rep) {
    df <- data.frame(df, raw[, sel+i-1])
  }
  raw <- df
}
names(raw) <- c('Wavelength', paste('sample', seq(colNum-1), sep = '.'))
```

## Initial figures

Here you can find the *unnormalized* figures and the peaks for your data.

```{r pressure, echo=FALSE}
#############################Visualization initial figure#########################
p <- ggplot(data = NULL, aes(x=wav))
for (i in 2:colNum) {
  print(i)
  randomwalk <- raw[,i]
  # Generate Data
  tops <- which.peaks(randomwalk, decreasing = FALSE)
  bottoms <- which.peaks(randomwalk, decreasing = TRUE)
  # Color functions
  cf.top <- grDevices::colorRampPalette("red")
  cf.bottom <- grDevices::colorRampPalette("blue")
  #plot(randomwalk, type = 'l', main = "Minima & Maxima\nVariable Thresholds")
  #lines(randomwalk, raw[,1])
  p <- p + geom_line(aes(x=wav, y=randomwalk), color="#0072B2")
  p <- p + geom_point(aes(x=wav[tops], y=randomwalk[tops]), color='red') + 
    geom_text(aes(x=wav[tops], y=randomwalk[tops], label = wav[tops]), nudge_y = 0.03, check_overlap = TRUE)
  p <- p + geom_point(aes(x=wav[bottoms], y=randomwalk[bottoms]), color='blue')
  p <- p + labs(x = 'Wavelength / nm', y = names(raw)[i])
  print(p)
  print(summ.peaks(raw[,i]))
}
raw.summ <- apply(raw[,2:colNum], 2, summ.peaks)
#print(raw.summ)
sink("00summ.txt"); print(raw.summ); sink()
#write.csv(raw.summ, file = "00summ.txt", row.names = FALSE, col.names = FALSE)
```

## Normalized data
If you turned the normalization on, here are the data after dividing by the normalization factor

```{r, echo=FALSE}
###############################Normalization##########################
if (do_norm) {
  newdf <- data.frame(raw[1])
  for (i in 2:colNum) {
    temp <- raw[,i]/norm_vec[i-1]
    newdf <- data.frame(newdf, temp)
  }
  names(newdf) <- names(raw)
  long.df <- melt(newdf,id.vars = 1)
  names(long.df) <- c('Wavelength', 'Sample', 'Intensity')
  q <- ggplot(data = long.df, aes(x = Wavelength, y= Intensity))
  q <- q + geom_line(aes(color = Sample))
  print(q)
  write.table(newdf, file = "00signal.txt", sep = ",", row.names = FALSE, col.names = FALSE)
}

```

