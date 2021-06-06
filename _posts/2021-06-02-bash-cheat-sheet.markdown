---
layout: single
title:  "Bash for Data Science"
date:   2021-06-02
categories: peripherals
comments: true
---

As a data professional, chances are you already know Python, one of the most powerful general purpose languages which has libraries to do almost anything conveniently. Why, then, to learn Bash and the Unix shell in general?
Some of the use cases I've found in my work:

* Using file system locally/remotely
* Monitoring system resources locally/remotely
* Inspecting data files remotely (locally I find it faster to just spin up a Python console with Pandas)
* Automating any jobs that make use of different command line tools
* Piping together complex preprocessing steps that involve compiling software or syncing with a db (possibly through Docker)
* Building the glue between query/preprocessing/train/deploy stages
* Piping together custom utilities that make use of tools you would use from CLI anyway (e.g Git)
* Makefile is often used as a poor man's DAG framework for data science, for example in the [cookiecutter](https://github.com/drivendata/cookiecutter-data-science/blob/master/%7B%7B%20cookiecutter.repo_name%20%7D%7D/Makefile) project

## Setup

Note that on Mac you're by default stuck with BSD utilities that are less powerful than [GNU](https://www.gnu.org/software/coreutils/manual/coreutils.html#split-invocation) ones. You can however do `brew install coreutils` and then use them with a `g` prefix, e.g. `split` becomes `gsplit`. 

You'll probably also want to update the Mac Bash version with `brew install bash`.

If any of the commands do not exist, then on Mac you can run `brew install command` usually.

On Mac, you probably want to install [iTerm2](https://iterm2.com/) instead of the default Terminal.

## Basics

Initiate scripts with either `#!/bin/bash` or `#!/usr/bin/env bash`. 

Put double quotes around `"$VARIABLE"`s to preserve the value.

`set -euox pipefail` is a nice default configuration (in scripts), which provides:

* raising an error in case any command in the script fails
* printing the commands as they are run
* raising an error in case of unset variables

Use `Ctrl+R` in terminal for searching through command history, `history` to see it completely. 

Use `info command` or `man command` or `help command` or `command --help` to get help with a command. `whatis command` also prints the first line description from the `man` manual.

## Aliases
An alias allows a string to be substituted for a word when it is used as the first word of a simple command. Aliases have many uses cases, but one of the main ones is for increasing productivity. You can create simple shortcuts for repetitive tasks such as 

connecting to a remote machine:
`alias remote="ssh user@bigawesomemachine.cloud`

connecting to a database:
`alias db="psql -h database.redshift.amazonaws.com -d live -U database_user -p 5439"`

activating a Conda environment:
`alias pyenv="source activate py_39_latest`

and even just pure laziness:
`alias jn="jupyter notebook"`

`alias repo="cd /Users/john/work/team_repo"`

`alias l="ls -lah"`


## Inspecting files

Use `tree` to list contents of directories in a tree-like format.

Read one end of a file

`head -n 10 file` or `tail -n 10 file`

Observe a dynamically changing file, such as a log

`tail -f file.log`

Find a file in the current (nested) directory

`find . -name data.csv`

Browse a .csv file where "," is the column separator

`cat data.csv |  column -t -s "," | less -S`

Get the value for a key from a plaintext configuration that is not necessarily sourcable in Bash
`sed -n 's/^MODEL_DATA_PATH = //p' model_conf.py`

You can do a Pandas-style value counts on a .csv column with `cut`, for example for the fifth column in a semicolon separated file

`cut -d ";" -f 5 data.csv | sort | uniq -c | sort`

Compare files with `diff file1 file2` or `cmp file1 file2` and directories with `diff <(ls directory1) <(ls directory2)`. 

Use `grep` to search files for a (regex) pattern. This command takes a lot of useful options, so better to read `info grep`. grep+cut makes for a poweful pattern to filter rows on a pattern and get the column you care about, for example to get the latest commit hash:

`git log -1 | grep '^commit' | cut -d " " -f 2`

`sort` and `uniq` can be used together to remove duplicate lines of code, for example to count the number of duplicate lines in a .csv file

`sort data.csv | uniq -d | wc -l`

## Control structures

Fast if-then-else:

`[[ "$ENV" == prod ]] && bash run_production_job.sh || bash run_test_job.sh`

Checking for file existence:
`[[ -f configuration.file ]] && bash run_training_job.sh || echo "Configuration missing"`

Numeric comparisons: 

`[[ "$MODEL_ACCURACY" -ge 0.8 ]] && bash deploy_model.sh || echo "Insufficient accuracy" | tee error.log`

Bash is a programming language, so it supports the regular `for`/`while`/`break`/`continue` structures. For example, to create backup files:

```for i in `ls .csv`; do cp "$i" "$i".bak ; done```

Definining functions in Bash is also straightforward and might be a useful alternative to aliases for more complex logic

```bash
view_map() {
    open "https://www.google.com/maps/search/$1,$2/"
}I
```

For example the above opens a Google maps browser window in Tallinn with `view_map 59.43 24.74`

Use `xargs` to execute a command for multiple arguments:
for example remove all local Git branches that have been merged to master.

`git branch --merged master | grep -v "master" | xargs git branch -d` 

## IO basics

Append to a file

`python server.py >> server.log`

Write stdout and stderr to a file and console

`python run_something.py 2>&1 | tee something.log`

Write all output to the void (discard it)

`python run_something.py > /dev/null 2>&1`


## Manipulating string

You can also generate multiple strings with brace expansion:
`mkdir /var/project/{data,models,conf,outputs}`

There are [a few](https://tldp.org/LDP/abs/html/string-manipulation.html) nifty string manipulation functionalities built into Bash such as

`${string//substring/replacement}` 

for string replace or lower to upper case using translation

`cat lower_case_file | tr 'a-z' 'A-Z'`.

However, there are much more powerful languages such `awk` and `sed` built in to handle any string manipulation. You can also use Perl or Python with regular expressions for string manipulation of arbitrary complexity. Most likely you'd still use `awk`/`sed` for simpler patterns (if it's simple enough to Google it in a few minutes).

To get started with regular expressions, what I've found useful is interactive tutorials such as [RegexOne](https://regexone.com/) and playgrounds such as [regexr](https://regexr.com/).

## Manipulating files

Ensure that a file exists:

```bash
if [[ ! -f files/necessary.file ]]; then
	mkdir -p files
	touch files/necessary.file
fi
```

Compressing and decompressing a directory

`tar -cvzf archive_name.tar.gz content_directory`

`tar -xvzf archive_name.tar.gz -C target_directory`


Merge two .csv files that have an identical index

`join file1.csv file2.csv`

Merge N .csv files with a separate header file

`cat header.csv file1.csv file2.csv ... > target_file.csv` 

Use `split` to separate a file into chunks, for example for a .csv file (keeping the header)

```bash
tail -n +2 file.csv | split -l 4 - split_
for file in split_*
do
    head -n 1 file.csv > tmp_file
    cat "$file" >> tmp_file
    mv -f tmp_file "$file"
done
```

Take a random sample from a large .csv file (this will load it in memory though):

`shuf -n 10 data.csv`
and if you need to keep the header but write a subsample to another file:

```bash
head -n 1 data.csv > data_sample.csv
shuf -n 10000 <(tail -n +2 data.csv) >> data_sample.csv
```

Change a file separator with `sed`
`sed 's/;/,/g' data.csv > data.csv`

Send a file to a remote machine

`scp /Users/andrei/local_directory/conf.py user@hostname:/home/andrei/conf.py`


## Managing resources and jobs

What is eating up all my disk space?
`ncdu` provides an interactive view. An alternative without additional installs is `du -hs * | sort -h`

What/who is slowing down the entire machine?
`htop` for the colorful version, `top` otherwise.

`uptime` - self explanatory.

Leave a long running model/script executing in the background on a server and write the outputs to `model.log`
`nohup python long_running_model.py > model.log &`
This also print the PID you can use to kill the process with, if it's been a while you can find the PID with `ps -ef | grep long_running_model.py`

Send a request with (nested) payload to a local Flask service

```bash
curl \
  -H 'Content-Type: application/json' \
  -X POST \
  -d '{"model_type": "neuralnet", "features": {"measurement_1": 50002.3, "measurement_2": -13, "measurement_3": 1.001}}' \
  http://localhost:5000/invoke
```

To kill jobs, `kill -9 PID` is the [most destructive](https://tldp.org/LDP/Bash-Beginners-Guide/html/sect_12_01.html#sect_12_01_02) option and impossible for jobs to ignore, but should be used as last resort. Start with Ctrl+C or `kill PID` (uses TERM signal by default).


## Variables

### Basics

Use `printenv` to see your environment variables. To set a permanent environment variable, add it to your `~/.bash_profile`:
`export MLFLOW_TRACKING_URI=http://mlflow.remoteserver`
Now you can access `os.environ["MLFLOW_TRACKING_URI"]` in Python.

Use `env` to run a command with a custom environment:

```bash
env -i INNER_SHELL=True bash
```

Use `local` to declare local scope variables in a function.

### Special parameters

There are a [few](https://www.thegeekstuff.com/2010/05/bash-shell-special-parameters/), but some useful ones

`$? $@` etc

`$?` - exit status of the last command
`$#` - number of positional arguments for a script
`$$` - process ID
`$_` - absolute path of the shell
`$0` - name of the script
`$!` - PID of the last background script

`[[ $? == 0 ]] && echo "Last command succeeded" || echo "Last command failed"`

### Variable manipulation

Some tricks you can do with variables: 

If parameter is not set, use a default value:
`${MODEL_DIR:-"/Users/jack/work/models"}`
or set it to another value:
`${MODEL_DIR:=$WORK_DIR}`
or raise an error with a message:
`${MODEL_DIR:?'No directory exists!'}`

## Command line arguments

`$#` - the number of command line arguments

You could always use the simple `$1`, `$2`, `$3` to use positional arguments passed together with a script.

For simple single letter named arguments, you can use the builtin `getopts`.

```bash
while getopts ":m:s:e:" opt; do
    case ${opt} in
        m) MODEL_TAG="$OPTARG"
        ;;
        s) DATA_START_DATE="$OPTARG"
        ;;
        e) DATA_END_DATE="$OPTARG"
        ;;
        \?) echo "Incorrect usage!"
        ;;
    esac
done

```

## Varia

Python code can be run inline, as a bridge between Bash and Python scripting

```bash
python -c 'from sklearn.svm import SVC; from sklearn.multiclass import OneVsRestClassifier; from sklearn.metrics import accuracy_score; from sklearn.preprocessing import LabelBinarizer; X = [[1, 2], [2, 4], [4, 5], [3, 2], [3, 1]]; y = [0, 0, 1, 1, 2]; classif = OneVsRestClassifier(estimator=SVC(random_state=0)); y_preds = classif.fit(X, y).predict(X); print(accuracy_score(y, y_preds))';
```

Another way to feed multi line input to a command would be using the here document, for example:

```bash
python <<HEREDOC
import sys
for p in sys.path:
  print(p)
HEREDOC
```

where `HEREDOC` acts as the EOF encoding and the script is executed only after the EOF has been met.

Use the `jq` command for working on the command line with JSON. [Examples](https://www.datascienceatthecommandline.com/2e/chapter-5-scrubbing-data.html#working-with-xmlhtml-and-json).

## Resources

[Bash Guide for Beginners](https://tldp.org/LDP/Bash-Beginners-Guide/html/) - Decent resource for an intro into scripting

[GNU Coreutils manual](https://www.gnu.org/software/coreutils/manual/coreutils.html) - Learn what you can do with default tools

[Data Science at the Command Line](https://www.datascienceatthecommandline.com/) - if you like the shell so much that you'd like to do EDA/modelling there :)

Thanks for some tips and tricks to [Mark Cowan](https://hackology.co.uk/) who can actually script in Bash, rather
than throwing together trivial utilities like me.

