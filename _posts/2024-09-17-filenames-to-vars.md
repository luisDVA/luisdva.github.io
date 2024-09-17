---
title: Filenames to variables 
excerpt: "Putting information recorded in filenames into the data rectangle"
tagline: "Iterative approach"
classes: wide
category: rstats
tags: 
  - gtcars
  - fs
  - walk2
  - filepath
  - file names
header: 
  overlay_image: /assets/images/featureFnames.png
  overlay_filter: "0.4"
---


Here's a brief approach for putting information recorded only in the file names of multiple tabular files into the data rectangle at the time of import.  

The need for this arose because it is fairly common for government agencies to split related data into multiple files for individual download, with each file for example: containing data for a specific year or region. However, this can make it difficult to combine all the separate files into a global dataset when important variables are not recorded explicitly in the data.

For this example, let's work with the 🚗 `gtcars` dataset from the gt package. This dataset is like **mtcars** but for very fancy and premium cars.

We'll group `gtcars` by multiple varibles, split the data into groups, and export each group to a separate csv file without the grouping variables but with this information in the file names. Then, we'll import all the files, splitting the file names and adding the relevant information back into the data rectangle. This will allow us to combine all the content into a single dataset.


##  📂 Create a folder with multiple csv files 

First, load everything we need.

{% highlight r %}
# load gt and various tidyverse pkgs
library(gt)      # CRAN v0.10.1 
library(dplyr)   # CRAN v1.1.4
library(purrr)   # CRAN v1.0.2
library(glue)    # CRAN v1.7.0
library(stringr) # CRAN v1.5.1
library(readr)   # CRAN v2.1.5
library(fs)      # CRAN v1.6.4

data(gtcars) # get the data 
{% endhighlight %}

Now, we can group the data by country of origin, year, and drive train and then a) split the data into a list of tibbles (one for each group), and b) use the group keys to build a vector of file names based on the grouping information used to split the data.


{% highlight r %}

# list of tibbles
gtcars_groups <- gtcars |> 
  group_split(ctry_origin, year, drivetrain)

# tibble of the grouping variables
gtgroups <- 
gtcars |> 
  group_by(ctry_origin, year, drivetrain) |> 
  group_keys() 
{% endhighlight %}

After minor changes like replacing spaces, the vector of filenames can be built with rowwise glueing of the values in the grouping variables.


{% highlight r %}
grpnames <- 
    gtgroups |> 
      rowwise() |> 
      mutate(gluedvars = glue_collapse(across(everything(), as.character), sep = "_")) |> 
      mutate(gluedvars= str_replace(gluedvars," ","-")) |> 
      pull(gluedvars)
{% endhighlight %}


🗋 Let's use this vector of filenames to name the list of tibbles, and then export each tibble to a separate csv file using `walk2` for its side effects.

{% highlight r %}
names(gtcars_groups) <- grpnames
gtcars_groups <- 
  gtcars_groups |> map(select,-c(ctry_origin,year,drivetrain))
# to disk  
  walk2(gtcars_groups,names(gtcars_groups),
        \(x,y) write_csv(x,paste0("carfiles/",y,".csv")))
{% endhighlight %}

We should now have in our working directory: a folder ("carfiles" in this example) full of csv files.

Now let's write a function that will strip the filename, split it into fragments (one for each variable), and add these fragments as values into the data rectangle as part of the import step. Using `read.csv` here to get around guessing and type conversion.


{% highlight r %}
filenameTovars <- function(filepath, var_names = NULL) {
  # strip the filename
  filename <- basename(filepath)
  filename <- tools::file_path_sans_ext(filename)
  
  # split the filename 
  filename_parts <- str_split(filename, "_", simplify = TRUE)
  
  # import
  data <- read_csv(filepath)
  
  # if no var_names provided
  if (is.null(var_names))  {
    var_names <- paste0("v", seq_along(filename_parts))
  }

  # filename fragments as new variables
  as_tibble(bind_cols(
    data,
    setNames(as.list(filename_parts), var_names) # a df that will be recycled
  ))
}
{% endhighlight %}

Nice little function above, it can take an optional vector with the names of the variables being added and if none are provided, it will use "v1", "v2", etc. Hard-coded "_" as separator here, but that could be an argument too.

For one of the files chosen at random, let's compare the output from reading the csv normally vs applying the new function with and without the vector of variable names.


{% highlight r %}
# read the csv normally
read_csv("carfiles/Italy_2015_rwd.csv")

# read with path only
filenameTovars("carfiles/Italy_2015_rwd.csv") 

# read with names for new columns
filenameTovars("carfiles/Italy_2015_rwd.csv",
        var_names = c("ctry_origin","year","drivetrain")) 


{% endhighlight %}

The csv file without the data from the filename.

{% highlight text %}
# A tibble: 6 × 12
  mfr   model trim  bdy_style    hp hp_rpm   trq trq_rpm mpg_c mpg_h trsmn
  <chr> <chr> <chr> <chr>     <dbl>  <dbl> <dbl>   <dbl> <dbl> <dbl> <chr>
1 Ferr… 458 … Base… coupe       597   9000   398    6000    13    17 7a   
2 Ferr… 458 … Base  converti…   562   9000   398    6000    13    17 7a   
3 Ferr… Cali… Base… converti…   553   7500   557    4750    16    23 7a   
4 Ferr… F12B… Base… coupe       731   8250   509    6000    11    16 7a   
5 Ferr… LaFe… Base… coupe       949   9000   664    6750    12    16 7a   
6 Lamb… Hura… LP 6… coupe       610   8250   413    6500    16    20 7a   
# ℹ 1 more variable: msrp <dbl>

{% endhighlight %}

Making this information explicit, but the new variables are named v1, v2, and v3.
{% highlight text %}
# A tibble: 6 × 15
  mfr   model trim  bdy_style    hp hp_rpm   trq trq_rpm mpg_c mpg_h trsmn
  <chr> <chr> <chr> <chr>     <int>  <int> <int>   <int> <int> <int> <chr>
1 Ferr… 458 … Base… coupe       597   9000   398    6000    13    17 7a   
2 Ferr… 458 … Base  converti…   562   9000   398    6000    13    17 7a   
3 Ferr… Cali… Base… converti…   553   7500   557    4750    16    23 7a   
4 Ferr… F12B… Base… coupe       731   8250   509    6000    11    16 7a   
5 Ferr… LaFe… Base… coupe       949   9000   664    6750    12    16 7a   
6 Lamb… Hura… LP 6… coupe       610   8250   413    6500    16    20 7a   
# ℹ 4 more variables: msrp <int>, v1 <chr>, v2 <chr>, v3 <chr>
{% endhighlight %}

Passing the names for the new columns:
{% highlight text %}
# A tibble: 6 × 15
  mfr   model trim  bdy_style    hp hp_rpm   trq trq_rpm mpg_c mpg_h trsmn
  <chr> <chr> <chr> <chr>     <int>  <int> <int>   <int> <int> <int> <chr>
1 Ferr… 458 … Base… coupe       597   9000   398    6000    13    17 7a   
2 Ferr… 458 … Base  converti…   562   9000   398    6000    13    17 7a   
3 Ferr… Cali… Base… converti…   553   7500   557    4750    16    23 7a   
4 Ferr… F12B… Base… coupe       731   8250   509    6000    11    16 7a   
5 Ferr… LaFe… Base… coupe       949   9000   664    6750    12    16 7a   
6 Lamb… Hura… LP 6… coupe       610   8250   413    6500    16    20 7a   
# ℹ 4 more variables: msrp <int>, ctry_origin <chr>, year <chr>,
#   drivetrain <chr>
{% endhighlight %}

Finally, we can use purrr to iterate through all the files in a folder and apply the custom function. Note the new recommended approach of using list_rbind rather than map_df.


{% highlight r %}
map(allpaths,\(x) filenameTovars(x,
  var_names = c("ctry_origin","year","drivetrain"))) |> 
  list_rbind() |> type_convert() 
{% endhighlight %}


Our output is essentially the same `gtcars` object we started with. 

{% highlight text %}
# A tibble: 47 × 15
   mfr   model    trim    bdy_style    hp hp_rpm   trq trq_rpm mpg_c mpg_h
   <chr> <chr>    <chr>   <chr>     <int>  <int> <int>   <int> <int> <int>
 1 Audi  R8       4.2 (M… coupe       430   7900   317    4500    11    20
 2 BMW   i8       Mega W… coupe       357   5800   420    3700    28    29
 3 Audi  RS 7     Quattr… hatchback   560   5700   516    1750    15    25
 4 Audi  S6       Premiu… sedan       450   5800   406    1400    18    27
 5 Audi  S7       Presti… hatchback   450   5800   406    1400    17    27
 6 Audi  S8       Base S… sedan       520   5800   481    1700    15    25
 7 BMW   6-Series 640 I … coupe       315   5800   330    1400    20    30
 8 BMW   M4       Base C… coupe       425   5500   406    1850    17    24
 9 BMW   M5       Base S… sedan       560   6000   500    1500    15    22
10 BMW   M6       Base C… coupe       560   6000   500    1500    15    22
# ℹ 37 more rows
# ℹ 5 more variables: trsmn <chr>, msrp <int>, ctry_origin <chr>,
#   year <dbl>, drivetrain <chr>

{% endhighlight %}

Hopefully others out there find this useful!
