# GNU Parallel Examples

+ [Video](https://yewtu.be/watch?v=qypUdm-IE9c)
+ [Article](https://omgenomics.com/parallel/)
+ [GNU Parallel Homepage](https://www.gnu.org/software/parallel/)

```sh
#!/usr/local/bin/bash

# Using GNU parallel painlessly
# maria.nattestad@gmail.com
# Jul 18, 2022

# https://omgenomics.com/parallel/
# https://www.gnu.org/software/parallel/
# https://yewtu.be/watch?v=qypUdm-IE9c

## 1) Basic examples
## --

parallel echo ::: A B C

# Get inputs from a command:
parallel echo ::: $(seq 10)

# Makes a sample file with numbers 1 to 100.
seq 100 > file.txt  

# Get inputs from a file:
parallel echo ::: $(cat file.txt)

# same as (using 4 colons):
parallel echo :::: file.txt

# You can get inputs from a pipe:
cat file.txt | grep "2" | parallel echo "hello"

# Every possible combination:
parallel echo ::: A B C ::: 1 2 3

# A 1
# A 2
# B 1
# A 3
# B 2
# B 3
# C 1
# C 2
# C 3

# Linked inputs:
parallel --link echo ::: A B C ::: 1 2 3

# A 1
# B 2
# C 3

## 2) Run a script in parallel
## --

echo '
JOB_ID=${1}
sleep 2
echo "Finished job #${JOB_ID}"
' > do_something.sh

# make it executable
chmod +x do_something.sh

# run this:
parallel ./do_something.sh ::: 1 2 3

# output: after 2 seconds have passed, all appear at once:
# Finished job #1
# Finished job #3
# Finished job #2

# You can also try adding –verbose to see when each job starts running.
parallel --verbose ./do_something.sh ::: 1 2 3

# ./do_something.sh 1
# ./do_something.sh 2
# ./do_something.sh 3
# Finished job #1
# Finished job #3
# Finished job #2

# And you can control how many jobs run at once with --jobs:
parallel --jobs 2 --verbose ./do_something.sh ::: 1 2 3

# Using --jobs 1 is essentially the same as a basic for loop.

# Try the same tricks with this script, which sleeps for the specified number of seconds:
echo '
sleep ${1}
echo "slept ${1} seconds."
' > do_something2.sh

chmod +x do_something2.sh

parallel ./do_something2.sh ::: 1 2 3

# output, now waiting 1 second in between each line:

# slept 1 seconds.
# slept 2 seconds.
# slept 3 seconds.

## 3) Multiple arguments:
## --
 
echo '
sleep ${1}
echo "${2} slept ${1} seconds"
' > do_something3.sh

chmod +x do_something3.sh

# parallel ./do_something3.sh ::: A B C ::: 1 2 3
# Whoops, teachable mistake here:

# 1 slept A seconds
# usage: sleep seconds
# 2 slept A seconds
# usage: sleep seconds
# 3 slept A seconds
# usage: sleep seconds
# 1 slept B seconds
# usage: sleep seconds
# 2 slept B seconds
# usage: sleep seconds
# 3 slept B seconds
# usage: sleep seconds
# 1 slept C seconds
# usage: sleep seconds
# 2 slept C seconds
# usage: sleep seconds
# 3 slept C seconds
# usage: sleep seconds

# This ran, for example, ./do_something3.sh A 1, so the script ran sleep A which
# is wrong, we wanted sleep 1. You can fix this by changing the order of
# arguments to ::: 1 2 3 ::: A B C, or you can specify where to insert the
# arguments – this is more flexible.

parallel ./do_something3.sh {2} {1} ::: A B C ::: 1 2 3
# A slept 1 seconds
# B slept 1 seconds
# C slept 1 seconds
# A slept 2 seconds
# B slept 2 seconds
# C slept 2 seconds
# A slept 3 seconds
# B slept 3 seconds
# C slept 3 seconds

# Much better.

## 4) Running parallel with a function
## --

# This is nice when you want something more complex than the small examples above
# but without writing a script.

# Try this: (This doesn't work for me in the ZSH shell (new Mac) though.)
function make_a_file_exported() {
    seq ${2} > numbers_${1}.txt
}
export -f make_a_file_exported
parallel --link make_a_file_exported {1} {2} ::: A B C ::: 10 20 30

# Otherwise you can use env_parallel:
# follow instructions the first time you run this to make it work.
source env_parallel.bash

function make_a_file_linked() {
    seq ${2} > numbers_env_${1}.txt
}
env_parallel --link make_a_file_linked {1} {2} ::: A B C ::: 10 20 30

# If neither of those work for you, google around for answers or just use scripts instead.

## 5) Practical tips for launching bioinformatics jobs
## --

# You can use parallel to orchestrate jobs for you on the cloud or on your lab/
# institution’s cluster. As an example, I’ve been using this a lot at work where
# we have a script that launches one stage at a time on Google Cloud, and then
# when a step finishes, it will take that output as the input for the next stage.
# It’s nice to use parallel to orchestrate this so I don’t have to open each of
# these orchestration scripts in different terminal tabs (or screen/tmx2 tabs).

# In this example, you have a runner_script.sh that might launch a job or a
# series of interdependent jobs on a cloud or cluster. We give it some input
# files and in this case I want to see the result at a few different filter
# values, so I get that combination of each input file with each of the 3 values
# of quality filter.

# you can make some fake input files for practice like this:
parallel touch "input_file_{1}.bam" ::: $(seq 10)

# Use a function to specify carefully how the runner script should be run:
function launch_runner_script() {
    INPUT=${1}
    Q_FILTER=${2}
    OUTPUT=${INPUT%.bam}.output.q${Q_FILTER}
    echo "runner_script.sh --input ${1} --output ${OUTPUT}.bam --filter ${Q_FILTER} 2>&1 > ${OUTPUT}.log"
}

echo "env parallel example 2"
env_parallel launch_runner_script {1} {2} ::: $(ls input_file*.bam) ::: 10 20 30

# You’ll notice a few things above:

#  1. We use the input name to determine the output name, removing the .bam
#     prefix and attaching that Q_FILTER value to differentiate the outputs. I
#     try to make this as clean as possible
#  2. We write a log file next to the output. ` 2>&1 >` means we’re capturing
#     both stderr and stdout and writing them to the same log file. I always make
#     sure to capture logs for each run individually so I can check on them along
#     the way and so I have a hope of debugging any problems that arise.
#  3. We echo the runner_script.sh command, this is always a good idea to do
#     first to check the output commands are what you expected before you proceed
#     (then you can remove the echo and quotes to run it for real). See the
#     output below:

# Output:

# runner_script.sh --input input_file_1.bam --output input_file_1.output.q30.bam --filter 30 2>&1 > input_file_1.output.q30.log
# runner_script.sh --input input_file_3.bam --output input_file_3.output.q10.bam --filter 10 2>&1 > input_file_3.output.q10.log
# runner_script.sh --input input_file_1.bam --output input_file_1.output.q20.bam --filter 20 2>&1 > input_file_1.output.q20.log
# runner_script.sh --input input_file_2.bam --output input_file_2.output.q20.bam --filter 20 2>&1 > input_file_2.output.q20.log
# runner_script.sh --input input_file_10.bam --output input_file_10.output.q30.bam --filter 30 2>&1 > input_file_10.output.q30.log
# runner_script.sh --input input_file_2.bam --output input_file_2.output.q10.bam --filter 10 2>&1 > input_file_2.output.q10.log
# runner_script.sh --input input_file_10.bam --output input_file_10.output.q10.bam --filter 10 2>&1 > input_file_10.output.q10.log
# runner_script.sh --input input_file_2.bam --output input_file_2.output.q30.bam --filter 30 2>&1 > input_file_2.output.q30.log
# runner_script.sh --input input_file_1.bam --output input_file_1.output.q10.bam --filter 10 2>&1 > input_file_1.output.q10.log
# runner_script.sh --input input_file_10.bam --output input_file_10.output.q20.bam --filter 20 2>&1 > input_file_10.output.q20.log
# runner_script.sh --input input_file_4.bam --output input_file_4.output.q20.bam --filter 20 2>&1 > input_file_4.output.q20.log
# runner_script.sh --input input_file_3.bam --output input_file_3.output.q20.bam --filter 20 2>&1 > input_file_3.output.q20.log
# runner_script.sh --input input_file_4.bam --output input_file_4.output.q10.bam --filter 10 2>&1 > input_file_4.output.q10.log
# runner_script.sh --input input_file_3.bam --output input_file_3.output.q30.bam --filter 30 2>&1 > input_file_3.output.q30.log
# runner_script.sh --input input_file_6.bam --output input_file_6.output.q20.bam --filter 20 2>&1 > input_file_6.output.q20.log
# runner_script.sh --input input_file_6.bam --output input_file_6.output.q10.bam --filter 10 2>&1 > input_file_6.output.q10.log
# runner_script.sh --input input_file_5.bam --output input_file_5.output.q30.bam --filter 30 2>&1 > input_file_5.output.q30.log
# runner_script.sh --input input_file_4.bam --output input_file_4.output.q30.bam --filter 30 2>&1 > input_file_4.output.q30.log
# runner_script.sh --input input_file_5.bam --output input_file_5.output.q20.bam --filter 20 2>&1 > input_file_5.output.q20.log
# runner_script.sh --input input_file_5.bam --output input_file_5.output.q10.bam --filter 10 2>&1 > input_file_5.output.q10.log
# runner_script.sh --input input_file_7.bam --output input_file_7.output.q10.bam --filter 10 2>&1 > input_file_7.output.q10.log
# runner_script.sh --input input_file_7.bam --output input_file_7.output.q20.bam --filter 20 2>&1 > input_file_7.output.q20.log
# runner_script.sh --input input_file_7.bam --output input_file_7.output.q30.bam --filter 30 2>&1 > input_file_7.output.q30.log
# runner_script.sh --input input_file_6.bam --output input_file_6.output.q30.bam --filter 30 2>&1 > input_file_6.output.q30.log
# runner_script.sh --input input_file_8.bam --output input_file_8.output.q30.bam --filter 30 2>&1 > input_file_8.output.q30.log
# runner_script.sh --input input_file_8.bam --output input_file_8.output.q10.bam --filter 10 2>&1 > input_file_8.output.q10.log
# runner_script.sh --input input_file_8.bam --output input_file_8.output.q20.bam --filter 20 2>&1 > input_file_8.output.q20.log
# runner_script.sh --input input_file_9.bam --output input_file_9.output.q30.bam --filter 30 2>&1 > input_file_9.output.q30.log
# runner_script.sh --input input_file_9.bam --output input_file_9.output.q10.bam --filter 10 2>&1 > input_file_9.output.q10.log
# runner_script.sh --input input_file_9.bam --output input_file_9.output.q20.bam --filter 20 2>&1 > input_file_9.output.q20.log
```
