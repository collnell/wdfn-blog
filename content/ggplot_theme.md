---
author: Brian Breaker
date: 2017-04-19
slug: ggplot2
draft: True
title: Plotting water level data in R
type: post
categories: Data Science
 




author_email: <bbreaker@usgs.gov>

tags:
  - R
  - geoknife

description: Using ggplot2 to create the start of a UGSG theme.
keywords:
  - R
  - geoknife



---
Get Started
===========

``` r
library(dataRetrieval)
library(ggplot2)
library(scales)
library(gtable)
library(grid)
library(gridExtra)
library(dplyr)
library(ggrepel)
library(ggmap)
library(ggpmisc)
library(mgcv)
library(RCurl)


# add USGS theme for plots
theme_USGS <-  function(base_size = 8){theme(
  plot.title = element_text (vjust = 3, size = 9,family="serif"),
  #plot.margin = unit (c(5.5, 5, 5.5, 5), "lines"),
  panel.border = element_rect (colour = "black", fill = F, size = 0.1),
  panel.grid.major = element_blank (),
  panel.grid.minor = element_blank (),
  panel.background = element_rect (fill = "white"),
  legend.background = element_blank(),
  legend.justification=c(0, 0),
  legend.position = c(0.1, 0.7),
  legend.key = element_blank (),
  legend.title = element_blank (),
  legend.text = element_text (size = 8),
  axis.title.x = element_text (size = 9, family="serif"),
  axis.title.y = element_text (vjust = 1, angle = 90, size = 9, family="serif"),
  axis.text.x = element_text (size = 8, vjust = -0.25, colour = "black",
                              family="serif", margin=margin(10,5,20,5,"pt")),
  axis.text.y = element_text (size = 8, hjust = 1, colour = "black",
                              family="serif", margin=margin(5,10,10,5,"pt")),
  axis.ticks = element_line (colour = "black", size = 0.1),
  axis.ticks.length = unit(-0.25 , "cm"),
  axis.ticks.margin = unit(0.5, "cm")
)}

# add function to draw ticks on plots
drawTicks <- function(plots, n = 1) {
  if(n == 1) {
    p1 <- ggplot_gtable(ggplot_build(plots))
    panel <-c(subset(p1$layout, name=="panel", se=t:r))
    rn <- which(p1$layout$name == "axis-b")
    axis.grob <- p1$grobs[[rn]]
    axisb <- axis.grob$children[[2]]  
    xaxis = axisb$grobs[[1]]  
    xaxis$y = xaxis$y - unit(0.25, "cm")  
    p1 <- gtable_add_rows(p1, unit(0, "lines"), panel$t-1)
    p1 <- gtable_add_grob(p1, xaxis, l = panel$l, t = panel$t, r = panel$r, name = "ticks")
    panel <-c(subset(p1$layout, name=="panel", se=t:r))
    rn <- which(p1$layout$name == "axis-l")
    axis.grob <- p1$grobs[[rn]]
    axisl <- axis.grob$children[[2]]  
    yaxis = axisl$grobs[[2]]
    yaxis$x = yaxis$x - unit(0.25, "cm")
    p1 <- gtable_add_cols(p1, unit(0, "lines"), panel$r)
    p1 <- gtable_add_grob(p1, yaxis, t = panel$t, l = panel$r+1, name = "ticks")
    p1$layout[p1$layout$name == "ticks", ]$clip = "off"
    maxWidth = unit.pmax(p1$widths[2:3])
    p1$widths[2:3] <- maxWidth
    g <- grid.arrange (p1, ncol=1)
  }
  else if(n > 1) {
    for(i in seq(1, n, 1)) {
      p0 <- ggplot_gtable(ggplot_build(get(plots[i])))
      panel <-c(subset(p0$layout, name=="panel", se=t:r))
      rn <- which(p0$layout$name == "axis-b")
      axis.grob <- p0$grobs[[rn]]
      axisb <- axis.grob$children[[2]]  
      xaxis = axisb$grobs[[1]]  
      xaxis$y = xaxis$y - unit(0.25, "cm")  
      p0 <- gtable_add_rows(p0, unit(0, "lines"), panel$t-1)
      p0 <- gtable_add_grob(p0, xaxis, l = panel$l, t = panel$t, r = panel$r, name = "ticks")
      panel <-c(subset(p0$layout, name=="panel", se=t:r))
      rn <- which(p0$layout$name == "axis-l")
      axis.grob <- p0$grobs[[rn]]
      axisl <- axis.grob$children[[2]]  
      yaxis = axisl$grobs[[2]]
      yaxis$x = yaxis$x - unit(0.25, "cm")
      p0 <- gtable_add_cols(p0, unit(0, "lines"), panel$r)
      p0 <- gtable_add_grob(p0, yaxis, t = panel$t, l = panel$r+1, name = "ticks")
      p0$layout[p0$layout$name == "ticks", ]$clip = "off"
      maxWidth = unit.pmax(p0$widths[2:3])
      p0$widths[2:3] <- maxWidth
      assign(paste0("p", i), p0, envir = .GlobalEnv)
    }
    forGetVec <- paste0("p", seq(1, n, 1))
    forGet <- list()
    for(newI in seq(1, n, 1)) {
      forGet[[newI]] <- get(forGetVec[newI])
    }
    g <- grid.arrange(grobs = forGet, ncol = 1)
  }
}

# modified a function found on stackoverflow to
# annotate ggplot with regression info and site number
lm_eqn <- function(m, site) {
  l <- list(a = format(coef(m)[1], digits = 2),
            b = format(abs(coef(m)[2]), digits = 2),
            r2 = format(summary(m)$r.squared, digits = 3),
            site = paste("USGS", site));

  if (coef(m)[2] >= 0)  {
    eq <- substitute(italic(y) == a + b~italic(x)*","~~italic(r)^2~"="~r2*","~~site,l)
  } else {
    eq <- substitute(italic(y) == a - b~italic(x)*","~~italic(r)^2~"="~r2*","~~site,l)    
  }

  as.character(as.expression(eq));                 
}
```

``` r
# vector of sites
sites <- c("345407091463801","345129091455801","345519091505401","345430091512301",
           "345427091524801","345415091505301","345402091502201","345413091493401",
           "345147091533301","345125091533301","345709091460701","345757091515401",
           "345753091543201","345541091491401")

# get site file data
info <- readNWISsite(sites)

# create a new data frame of just site number and altitude
infoNew <- info[,c(2,20)]

# get water levels
gwLevs <- readNWISgwl(sites, startDate = "", endDate = "", convertType = TRUE, tz= "UTC")
```

Display site info
=================

<iframe seamless src="/static/ggplot2/ggplot2/index.html" width="100%" height="500">
</iframe>

``` r
# merge water levels and alitudes
gwLevs <- merge(x = gwLevs, y = infoNew,
                by = "site_no",
                all.x = TRUE)

# add msl value to data frame of readings
gwLevs$watAlt <- gwLevs$alt_va - gwLevs$lev_va
```

Plot a one site
---------------

``` r
# get plots for individual sites in a for loop

j <- 1

# subset water level data by site number
subDF <- filter(gwLevs, site_no == sites[j])
# create the plot with a gam regression line
newPlot <- ggplot(data = subDF, aes(x = lev_dt, y = watAlt)) +
  geom_point() +
  scale_y_continuous() +
  stat_smooth(method = "lm") +
  annotate("text", x = mean(subDF$lev_dt), y = 0.995*max(subDF$watAlt),
           label=lm_eqn(lm(watAlt ~ lev_dt, subDF), sites[j]), parse=TRUE,
           family = "serif", size = 3) +
  labs(y = "Water Level Altitude,\nin feet above NAVD88", x = "Date") +
  theme_USGS()
```

    ## Warning: `axis.ticks.margin` is deprecated. Please set `margin` property of
    ## `axis.text` instead

``` r
# print the plot with ticks on all 4 sides
drawTicks(newPlot)
```

{{< figure src="/static/ggplot2/unnamed-chunk-5-1.png" title="TODO" alt="TODO" >}}

``` r
  # save the plot with ticks on all 4 sides as pdf
```

Map the sites
=============

Bonus round, make a pretty map of the sites.

``` r
# create a map of the site locations
# create a vector for the bounding box
forLocation <- c(min(info$dec_long_va) - 0.25, min(info$dec_lat_va) - 0.25,
                 max(info$dec_long_va) + 0.25, max(info$dec_lat_va) + 0.25)

# create a base map using the lat/long coordinates in the spatialPointsDataFrame
baseMap <- ggmap::get_map(location = forLocation, maptype = "terrain")
```

    ## Warning: bounding box given to google - spatial extent only approximate.

``` r
# add the sites to the baseMap and store it
map <- ggmap(baseMap) +
  geom_point(data = info, aes(x = dec_long_va, y = dec_lat_va),
             alpha = 1, fill = "red", pch = 21, size = 3) +
  labs(x = "Longitude", y = "Latitude") +
  geom_text_repel(data = info, aes(x = dec_long_va, y = dec_lat_va, label = site_no),
                  family = "serif") +
  theme_USGS()
```

    ## Warning: `axis.ticks.margin` is deprecated. Please set `margin` property of
    ## `axis.text` instead

``` r
# add the ticks to the map and draw it
drawTicks(map)
```

{{< figure src="/static/ggplot2/unnamed-chunk-6-1.png" title="TODO" alt="TODO" >}}

Questions
=========

Please direct any questions or comments on `geoknife` to: <https://github.com/USGS-R/geoknife/issues>
