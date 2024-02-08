---
layout: post
title:  "Dynamic Pandas Data Slicing"
author: robweber
categories: data-analytics work coding
tags: python yaml
---

Recently I wanted to modify a Python data analysis script I use regularly at work into more of a general purpose tool. The script uses the [Pandas Python library][pandas] to slice up data in CSV files based on some specific time periods. The issue I had is that these time periods changed often, and I was sick of mucking about in the script to constantly tweak them. I needed a way to specify the time periods at runtime instead of hard coding them.

This turned out to be more complicated than I envisioned so I figured I'd document it all for anyone interested (most likely myself in a year when I forget). __Note:__ this post is not a primer on Pandas, or data slicing, if you want more info on that [check the Pandas documentation](https://pandas.pydata.org/docs/user_guide/indexing.html).

<!--more-->

{% include toc %}

## The Problem

The project I had going involved a series of CSV files that contained some measurement values along with the date and time they were measured. For each day of the month there are a full day's worth of interval measurements - and I had years of them. Essentially they looked like this:

```
id, value, start-date, start-time, end-date, end-time
1, 1.2, 04-30-2023, 00:00, 04-03-2023, 00:15
1, 1.5, 04-30-2023, 0:15, 04-03-2023, 00:30
```

Using the data analysis library, Pandas, my script would read in these values as [DataFrames](https://pandas.pydata.org/docs/user_guide/dsintro.html#dataframe) so that the measurements could be summed up based on time periods. These time periods could be as simple as the same hours every day or a selection of hours on weekdays but a different selection over weekends. In the beginning I was hardcoding these time periods using normal Pandas syntax to slice the data frames.

```
# 5am - 3pm and 8pm every day as well as 4pm-7pm Sat and Sun
off_peak = forward_df[((forward_df['start-time'].str.startswith("06:")) | (forward_df['start-time'].str.startswith("07:")) |
                             (forward_df['start-time'].str.startswith("08:")) | (forward_df['start-time'].str.startswith("09:")) |
                             (forward_df['start-time'].str.startswith("10:")) | (forward_df['start-time'].str.startswith("11:")) |
                             (forward_df['start-time'].str.startswith("12:")) | (forward_df['start-time'].str.startswith("13:")) |
                             (forward_df['start-time'].str.startswith("14:")) | (forward_df['start-time'].str.startswith("15:")) |
                             (forward_df['start-time'].str.startswith("20:")) | (forward_df['start-time'].str.startswith("05:"))) |
                             (((forward_df['start-time'].str.startswith("16:")) | (forward_df['start-time'].str.startswith("17:")) |
                             (forward_df['start-time'].str.startswith("18:")) | (forward_df['start-time'].str.startswith("19:"))) &
                             ((forward_df['day-of-week'] == 'Sat') | (forward_df['day-of-week'] == 'Sun')))]
print(f"Off Peak: {off_peak['value'].sum()}")
```

If I wanted to see what happened by including a different hour in the sum, the script had to be manually adjusted. This was further compounded in that I needed to make sure all measurements were accounted for so each day had several different "time slices". Adding time to one meant reducing it from another.

## Building Conditionals

My goal was the make the script more generic so I could pass in the rules for each time period at runtime and have the script craft the conditionals for slicing the data dynamically. This way I could create different configurations and pass them in via a configuration file. I wasn't quite sure at first how to dynamically construct a conditional expression in Pandas so I did some searching. Eventually I found an example in [this blog post](https://www.dunderdata.com/blog/selecting-subsets-of-data-in-pandas-part-2).

```
# variable 'df' is an already loaded data frame
criteria_1 = df['score'] >= 5
criteria_2 = df['ans_name'] == 'Scott Boston'
criteria_all = criteria_1 & criteria_2

df[criteria_all]

```

This proved you could assign the conditions to a variable and use them in a selection statement. As a test I modified part of the time slicing code from above to use a function for generating the conditional expression. This still hardcoded the values, but proved that creating the expression at runtime would work.

```
def process_rule(rules, df):
    # must be at least one hour in the period
    result = (df['start-time'].str.startswith(f"{rules['start-time'][0]}:"))

    # add the rest of the times - splitting with OR (|)
    for time in rules['start-time'][1:]:
        result = result | (df['start-time'].str.startswith(f"{time}:"))

    return result

off_peak_conditional = create_conditional({"start-time": ["05", "06", "07", "08", "09", "10", "11", "12", "13", "14", "15"]}, forward_df)
off_peak = forward_df[off_peak_conditional]
print(f"Off Peak: {off_peak['value'].sum()}")
```

### Types of Conditions

The simple function I made only had one type of rule, slicing by hours of the day. I needed to have a bit more flexibility than that so I envisioned a few different types of rules:

1. Hours Of The Day - select times within the hours specified
2. Include Days - include specific days
3. Exclude Days - exclude specific days

The three different rulesets would be AND (&) linked together so I could create selections like "hours 16, 17, 18, 19 __AND__ only on Saturdays". The complex selection rules from the original code above would be expressed as an array of dictionary objects:

```
[
  # time slice 1 (5am - 3pm and 8pm every day)
  {
    "start-time": [
      "05", "06", "07", "08", "09", "10", "11", "12", "13", "14", "15", "20"
    ]
  },
  # time slice 2 - 4pm-7pm Sat and Sun only
  {
    "start-time": [
      "16", "17", "18", "19"
    ],
    "day-of-week": [
      "Sat", "Sun"
    ]
  }
]
```

Using this rulesets I could create a series of conditions that mirrored the time periods I needed to select data on.

## Time Period Configurations

The `process_rule` function created conditionals that Pandas data frames could use to slice up data. So far so good. As things stood I still had to hard code the values, only as Python lists and dictionary objects instead. To reach my desired end state I had to pass in these rules at runtime via a configuration file instead of putting them within the script. To do this I turned to configuration syntax I've used often - [YAML][yaml]. Not only is it easy to read, but Python makes it very easy to convert YAML into Python lists and dictionaries. Modifying the same ruleset again, it would look like this expressed in YAML:

```
time_periods:
  - name: "Off Peak"
    conditions:
      # time period 1
      - start-time:
        - "05"
        - "06"
        - "07"
        - "08"
        - "09"
        - "10"
        - "11"
        - "12"
        - "13"
        - "14"
        - "15"
        - "20"
      # time period 2
      - start-time:
         - "16"
         - "17"
         - "18"
         - "19"
        day-of-week:
          - "Sat"
          - "Sun"
```

The YAML would be read in at runtime from a specified file, and then parsed using the `process_rule` function. No more hardcoding the time periods.

### Time Period Verification

Looking at the time period syntax I realized it could be messed up fairly easily. It's not super complicated but nesting the lists properly and making sure to use valid selection syntax could get challenging. To combat this I went a little overboard and added a syntax verification when loading the file. To do this I used the [Cerebrus library][cerebrus]. Cerebrus allows you to define some validation rules that define what kinds of data and key/value pairs should exist in a set of data. In the conclusion is a link to the full schema but as an example of the power of Cerebrus, consider the following ruleset.

```
valid_times:
  allowed: &valid_times
    - "00"
    - "01"
    - "02"
    - "03"
    - "04"
    - "05"
    - "06"
    - "07"
    - "08"
    - "09"
    - "10"
    - "11"
    - "12"
    - "13"
    - "14"
    - "15"
    - "16"
    - "17"
    - "18"
    - "19"
    - "20"
    - "21"
    - "22"
    - "23"
```

This reusable set of values defines what strings will be considered valid time entries. When defining a time period if I accidentally write "24" instead of "00", or "1" instead of "01", this validation will catch it. The data in the CSV files is in a very specific format so trying to slice data on a non-existent character set will end with an incorrect result. Validation of the rules definitely took up extra time to implement but should ultimately serve as a check to ensure a proper result.

## Conclusion

Throwing everything together I was able to get the script working as I wanted. This will greatly reduce the amount of time spent changing values as I run this over different time periods for analysis. While specific to the use case I needed, the idea of dynamically creating conditions for data slicing could be used in a variety of ways. It would be very easy to add new slice conditions and extend the ruleset to slice data in a variety of different ways. I plan to revisit this idea for other projects in the future.


### Demo Project

For the sake of brevity I did leave out a lot of details. Specifically the loading of the Cerebrus library and final conditional rule processing functions. Unfortunately I can't share my working script as this was a work project, but I did create a heavily stripped down version with the key concepts as a full working example. These are available within a [demo project Gist on Github][demo-project].

Looking at the Gist there are a series of files to get the demo working.  

1. `analyze.py` - this is the main file that contains all the logic discussed in this post. This script will sum the values given the time periods in a series of CSV files.
2. `requirements.txt` - the Python libs required. Pandas should be installed as well but this is often OS specific
3. `schema.yaml` - the Cerebrus validation schema for the time period file
4. `test_data.csv` - some test data
5. `times.yaml` - some time periods to slice the data

To get this all working create a directory and put all the files except the `test_data.csv` in the root. Create a new directory `data/` and put the CSV file in there. Make sure you install the required libraries and run the script with:

```
python3 analyze.py -d data/
```

Once it's all working modify to suit your own needs!

## Links

* [Pandas][pandas] - Data analysis and manipulation library for Python
* [YAML][yaml] - human readable serialization language, commonly used for config files
* [Cerebrus][cerebrus] - a powerful and lightweight data validation tool for Python
* [Demo Project][demo-project] - a simple demo project showcasing these ideas

[pandas]: https://pandas.pydata.org/
[yaml]: https://en.wikipedia.org/wiki/YAML
[cerebrus]: https://docs.python-cerberus.org/
[demo-project]: https://gist.github.com/robweber/9e8bef1ccec45a97f82a5896ef71e97b
