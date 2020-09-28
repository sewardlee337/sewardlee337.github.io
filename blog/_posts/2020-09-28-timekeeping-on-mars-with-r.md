---
layout:     post
title:      Timekeeping on Mars with R
date:       2020-09-28
summary:    How to implement NASA's Mars24 Sunclock Algorithm in R for Earth-to-Mars time conversions.
categories: [astronomy]
tags: [astronomy, r]
---

<img src = "/assets/images/nasa-E7q00J_8N7A-unsplash.jpg">

My fascination with timekeeping on the Red Planet began with one of my favorite movies: 2015's sci-fi flick [_The Martian_](https://youtu.be/ej3ioOneTy8). In the scenes where Matt Damon's plucky astronaut character [speaks directly to the camera](https://www.youtube.com/watch?v=IDnUUJqdg-w), there's an indicator on the  corner of the screen that displays how many Martian-days has elapsed since his mission began.

This made me wonder: If timezone conversions are [such a massive nuisance on Earth](https://xkcd.com/1883/), how do you figure out the time on a completely different planet? Turns out that with the help of some resources provided by NASA, it's relatively straightforward to convert Earth-time to Mars-time using R.

## How Does Martian Time Work?

Dr. Michael D. Allison of NASA's Goddard Institute for Space Studies [published a paper in 1997](https://agupubs.onlinelibrary.wiley.com/doi/abs/10.1029/97GL01950) that spells out a methodology for finding the time on Mars. Additional information can also be found in Dr. Allison's 1998 Science Brief ["Telling Time on Mars"](https://www.giss.nasa.gov/research/briefs/allison_02/) and the [Wikipedia page on Martian timekeeping](https://en.wikipedia.org/wiki/Timekeeping_on_Mars).

What's important to know is that:
* A Martian day ("sol") is 39 minutes, 35.2 seconds longer than the terrestrial day of 24 hours
* A Martian year is 1.881 Earth years, or 668.59 sols
* Like a terrestrial day, a sol can be divided into 24 hours

Dr. Allison's became the basis for the [Mars24 Sunclock](https://www.giss.nasa.gov/tools/mars24/): a program developed in Java that keeps track of the time on Earth and Mars. Below, we're going to replicate the Mars24 algorithm in R.

## Implementing the Mars24 Sunclock Algorithm in R

NASA has posted [step-by-step instructions](https://www.giss.nasa.gov/tools/mars24/help/algorithm.html) on how to implement the Mars24 algorithm. This makes implementing it using R pretty straightforward.

First, let's import the packages that we'll need. The [**lubridate**](https://lubridate.tidyverse.org) package contains some useful datetime-conversion tools, and [**dplyr**](https://dplyr.tidyverse.org) allows us to use the pipe operator (`%>%`) in our code.

```r
library(dplyr)
library(lubridate)
```

For convenience, I coded up some trigonometric functions that accept degrees rather than radians as arguments. This allows the rest of the code to follow the worked examples on the NASA webpage more closely.

```r
# The following are trig functions that accept degrees rather
# than radians as arguments.

sin_deg <- function(x) {
     return(sin(x * pi / 180))
}

cos_deg <- function(x) {
     return(cos(x * pi / 180))
}

tan_deg <- function(x) {
     return(tan(x * pi / 180))
}

asin_deg <- function(x) {
     return(asin(x * pi / 180))
}

acos_deg <- function(x) {
     return(acos(x * pi / 180))
}

atan_deg <- function(x) {
     return(atan(x * pi / 180))
}
```

The following is a step-by-step implementation of the algorithm as described on NASA's webpage, bundled into a function called `Mars24()`.

```r
# Mars24 algorithm

Mars24 <- function(millis) {


     ## A-2: Convert millis to Julian Date (UT)

     jdUT <- 2440587.5 + (millis / (8.64 * 1e7))


     ## A-3: Determine time offset from J2000 epoch (UT)

     epoch.J2000 <- ymd_hms('2000-01-01 00:00:00')

     T <- ifelse(as_datetime(millis/1000) < epoch.J2000,
                    (jdUT - 2451545.0) / 36525, 0)


     ## A-4: Determine UTC to TT conversion

     tt.minus.ut <- 64.184 + (59 * T) - (51.2 * T^2) - (67.1 * T^3) - (16.4 * T^4)


     ## A-5: Determine Julian Date (TT)

     jdTT <- jdUT + (tt.minus.ut / 86400)


     ## A-6: Determine time offset from J2000 Epoch (TT)

     deltaTJ2000 <- jdTT - 2451545.0


     ## B-1: Determine Mars mean anomaly

     M <- 19.3871 + (0.52402073 * deltaTJ2000)


     ## B-2: Determine angle of Fiction Mean Sun

     alphaFMS <- 270.3871 + (0.524038496 * deltaTJ2000)


     ## B-3: Determine perturbers

     ### Note use of custom trig functions. Default trig functions in
     ### R only accept arguments in radians.

     alpha <- c(0.0071, 0.0057, 0.0039, 0.0037, 0.0021, 0.0020, 0.0018)
     tau <- c(2.2353, 2.7543, 1.1177, 15.7866, 2.1354, 2.4694, 32.8493)
     phi <- c(49.409, 168.173, 191.837, 21.736, 15.704, 95.528, 49.095)

     PBS <- sum(alpha * cos_deg(((0.985626 * deltaTJ2000 / tau) + phi)))


     ## B-4: Determine Equation of Center

     v.minus.M <- (10.691 + 3.0 * 1e-7 * deltaTJ2000) * sin_deg(M) +
          0.623 * sin_deg(2 * M) + 0.050 * sin_deg(3 * M) + 0.005 *
          sin_deg(4 * M) + 0.0005 * sin_deg(5 * M) + PBS


     ## B-5: Determine aerocentric solar longitude

     Ls <- alphaFMS + v.minus.M


     ## C-1: Determine Equation of Time

     EOT <- 2.861 * sin_deg(2 * Ls) - 0.071 * sin_deg(4 * Ls) +
          0.002 * sin_deg(6 * Ls) - v.minus.M


     ## C-2: Determine Coordinated Mars Time (ie Airy Mean Time)

     MTC <- (24 * (((jdTT - 2451549.5) / 1.0274912517) + 44796.0 - 0.0009626)) %% 24


     ## Return dataframe

     equation <- c('A-1', 'A-2', 'A-3', 'A-4', 'A-5', 'A-6', 'B-1', 'B-2',
                   'B-3', 'B-4', 'B-5', 'C-1', 'C-2')

     description <- c('Get a starting Earth time in millis',
                      'Convert millis to Julian Date (UT)',
                      'Determine time offset from J2000 epoch (UT)',
                      'Determine UTC to TT conversion',
                      'Determine Julian Date (TT)',
                      'Determine time offset from JS2000 epoch (TT)',
                      'Determine Mars mean anomaly',
                      'Determine angle of Fiction Mean Sun',
                      'Determine perturbers', 'Determine Equation of Center',
                      'Determine aerocentric solar longitude',
                      'Determine Equation of Time',
                      'Determine Coordinated Mars Time')

     value <- c(millis, jdUT, T, tt.minus.ut, jdTT, deltaTJ2000, M, alphaFMS,
                PBS, v.minus.M, Ls, EOT, MTC)

         output <- data.frame(equation = equation, description = description,
                               value = value)
         return(output)
}
```

The function `Mars24()` takes the time on Earth in [Unix time-format, at millisecond-level precision](https://www.geeksforgeeks.org/java-8-clock-millis-method-with-examples/). It returns a dataframe, where each row represents a step calculation listed in NASA's worked example.

We can technically use `Mars24()` as-is. But to make it a bit more friendly and allow room for future extension, let's wrap `Mars24()` in a second function: `earth2mars_convert()`. This will be function that the user of the algorithm will touch. Notice that there's an additional `verbose` argument that allows users to toggle whether they want a printout listing all the steps in the calculations.

```r
# User-friendly wrapper for Mars24 algorithm

earth2mars_convert <- function(millis, verbose=FALSE) {
  calculations <- Mars24(millis)

  if(verbose==TRUE) {
    print(calculations)
  }

  return(calculations$value[13])
}
```

To check if our implementation of the algorithm works, first convert a date to Unix time with millisecond-level precision. Here, let's use the date used in NASA's worked example: midnight of January 6, 2000.

```r
# Example arguments

datetime = '2000-01-06 00:00:00'
tzone = 'UTC'

example_datetime <- datetime  %>%
  ymd_hms(tz = tzone) %>%
  as.integer() * 1000
```

Then, call the function `earth2mars_convert()`.

```r
earth2mars_convert(example_datetime)
```

The function `earth2mars_convert()` should return: `23.99425`.

23.99425 hours, or 23:59:39, was the Mean Solar Time at Mars's prime meridian during midnight of January 6, 2000 (UTC) on Earth.
