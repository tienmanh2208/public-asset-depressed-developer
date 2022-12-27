# Selecting the right jobs

## Describing problem

Imagine you are a freelancer, and you get a lot of jobs from website, but you only can do one job at a time. Each job pays the same fee, and you want to get as much money as possible. So, you have to find the largest possible set of jobs such that no two of them conflict with each other.

You can check some examples bellow for more details. Assume you have the following jobs and the timeline of the jobs in the image.

**Example 1:**

<p align="center">
    <img src="https://raw.githubusercontent.com/tienmanh2208/public-asset-depressed-developer/master/posts/algorithm/select_right_jobs/example_list_jobs.png" alt="drawing"/>
</p>

The answer for this example might be (A, D, G, I) or (A, E, G, I). You can see that there is no other possible set that have more than 4 jobs such that no two of them conflict with each other.

**Example 2:**

<p align="center">
    <img src="https://raw.githubusercontent.com/tienmanh2208/public-asset-depressed-developer/master/posts/algorithm/select_right_jobs/example_2.png" alt="drawing"/>
</p>

The answer for this example might be (B, D, E, F, G).

**Example 3:**

<p align="center">
    <img src="https://raw.githubusercontent.com/tienmanh2208/public-asset-depressed-developer/master/posts/algorithm/select_right_jobs/example_3.png" alt="drawing"/>
</p>

The answer for this example might be (C, F).

## Generalize the problem

*Input*: A set I of n intervals on the line.

*Output*: What is the largest subset of non-overlapping intervals which can be selected from I.

## Solutions

### Exhaustive solution

We will try all possible cases, and return the case that has the largest size.

The algorithm might be like this:

$ExhaustiveSolution(I)$
&emsp;$j = 0$
&emsp;$S_{max} = 0$
&emsp;For each of the $2^n$ subsets $S_i$ of intervals $I$
&emsp;&emsp;If ($S_i$ is mutually non-overlapping) and $(size(S_i) > j)$
&emsp;&emsp;&emsp;$j = size(S_i)$
&emsp;&emsp;&emsp;$S_{max}=S_i$
&emsp;Return $S_{max}$
