---
layout: post
title: Parsing config files in bash using awk
date: 2020-05-17 00:00:00 +02:00
categories: SoWhatIsTheSolution
tags: [Linux, Bash, Awk]
fullview: true
description: Using config files in bash scripts can be quite helpful and a nightmare, awk to the rescue
author:
  name: Christian
  email: webmaster@christiansaga.de
---

Ever so often I get to some sort of complex bash script and find, that a configuration or ini file would make sense, to control the script an make it more flexible. However bash does not bring nice libraries to parse config files, e.g. like python.

In essence two things are needed to write a config file parser:
1. good trim of the config option, as you can have
* \<option>=\<value>
* \<option> = \<value>
* \<option> =\<value>
* and so on...

2. structured way to move the config to usable variables within bash.

### So what is the solution

First lets take a look on how to analyze a config file line. It is normally setup like: ```<option><delimiter><value>```

To parse this, we need to separate the option and value by the delimiter. For this ```awk``` is perfect to use as it can seperate a line with a field separator in parts and store each in a variable.

Additionally we can eliminate spaces around the delimiter to make the config parser more robust. However we need to do that left and right of the delimiter, to only trim the leading and trailing spaces of each side. Just eliminating spaces in the whole string would delete spaces in configurations as well.
Using the delimiter ```=``` this would look like the following for the left side (for right side change ```$1``` to ```$2```).

```
awk -F'[=]' '{gsub("^\\s+|\\s+$", "", $1); print $1}' <<< $line
```
So what does this do?
* ```-F[=]``` sets the delimiter to ```=```
* ```gsub("^\\s+|\\s+$", "", $1);``` starting starting from the first letter (```^```) replace 1 or more spaces at beginning and end (```\\s+|\\s+$```) with an empty string (```""```). ```$1``` use the left part of the delimiter for the gsub command
* ```print $1``` as ```awk``` automatically splits with the delimiter, print the first part (left side of the config line)
* ```<<< $line``` is the feeded line of our config. Within bash you can easily loop through a file line by line with the ```read``` command.

Now that we have the basics, we can write a parser function with it.

An example within a script:
{% highlight bash %}
#!/bin/bash

declare -A settings_map

function read_config () {
    while IFS= read -r line; do
        case "$line" in
            ""|\#*)
                # do nothing on comments and empty lines
                ;;
            *)
                # parse config name and value in array
                settings_map[$(awk -F'[=]' '{gsub("^\\s+|\\s+$", "", $1); print $1}' <<< $line)]=$(awk -F'[=]' '{gsub("^\\s+|\\s+$", "", $2);print $2}' <<< $line)
                ;;
        esac
    done < $1
    echo ${settings_map[*]}
}

# call fuction with the config file to parse
read_config parser-test.conf

# show that it is working and picking "option" from the array
echo ${settings_map[option]}
{% endhighlight %}

This would read a config file like this (or with some spaces around the ```=```):
```
# parser-test.conf
# Use option=value
#
type=value_type
option = value_option

test =value_test
spaces    = value space
```
It reads line by line, remove leading and trailing spaces on the left and right site of the config line, then store it into the settings_map array. The array can be used within the rest of the bash script.

**Important:** This function uses a globally declared array. Handing back an array from a function is not trivial. You could create and return a string, then initialize an array afterwards. However I feel that to be a bit cumberstone and not adding to the functionality.
