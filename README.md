# CodRep: Machine Learning on Source Code Competition

CodRep is a machine learning competition on source code data.
The goal of the competition is provide different communities (machine learning, software engineering, programming language) with a common playground to test and compare ideas.
The competition is designed with the following principles:

1. There is no specific background or skill requirements on program analysis to understand the data.
2. The systems that use the competition data can be used beyond the competition itself. In particular, there potential usages in the field of automated program repair.   

To take part to the competition, you have to write a program which predicts where to insert a specific line into a source code file.
In particular, we consider replacement insertions, where the new line replaces an old line, such as

```diff
public class test{
  int a = 1
-  int b = 0.1
+  double b = 0.1
}
```

More specifically, the program  takes as input a set of pairs (source code line, source code file), and outputs, for each pair,  the predicted line number of the line to be replaced by in the initial source code file.

The competition is organized by KTH Royal Institute of Technology, Stockholm, Sweden. The current organization team is Zimin Chen and Martin Monperrus.

## The winners

Hall of fame:

| Data | Name | Score Visible | Score Hidden | Link |
| --- | --- |--- | --- |--- |
| ... | ... | ... | ... |--- |

The scores are computed on the dataset present in this repository, as well as on an hidden dataset. The score on the hidden dataset is used for the final ranking.

What the participants get?

1. All participants get their name in the CodRep hall of fame
1. All participants will be invited to present their solutions at a physical workshop with proceedings.

What the winner gets?

1. She gets the ultimate CodRep fame
2. She receives nice KTH goodies by post
4. Her solution is invited to be part of the futuristic program repair bot designed and implemented at KTH


## Data Structure and Format

The provided data are in `Datasets/.../Tasks/*.txt`. The txt files are meant to be parsed by competing programs. Their format is as follows, each file contains:
```
{Code line to insert}
\newline
{The full program file}
```

For instance, let's consider this example input file, called `foo.txt`.
```java
double b = 0.1

public class test{
  int a = 1
  int b = 0.1
}
```
In this example, `double b = 0.1` is the code line to be added somewhere in the file in place of another line.

For such an input, the competing programs output for instance `foo.txt 3`, meaning replacing line 3 (`int b = 0.1`) with the new code line `double b = 0.1`.

To train the system, the correct answer for all input files is given in folder `Datasets/.../Solutions/*.txt`,  e.g. the correct answer to `Datasets/Datasets1/Tasks/1.txt` is in `Datasets/Datasets1/Solutions/1.txt`

## Data provenance

The data used in the competition is taken from real commits in open-source projects.
For a number of different projects, we have analyzed all commits and extracted all the one line replacement changes.
We have further filtered the data  based on the following criteria:

* Only source code files are kept (Java files in dataset00)
* Comment-only changes are discarded (e.g. replacing `// TODO` with `// Fixed`)
* Inserted or removed lines are not empty lines, and are not space-only changes

The datasets used in this competition are from:

| Directory | Original dataset | Published paper |
| --- | --- |--- |
| Dataset1/ | [github](https://github.com/monperrus/real-bug-fixes-icse-2015/) | [*An Empirical Study on Real Bug Fixes (ICSE 2015)*](http://stap.sjtu.edu.cn/images/8/86/Icse15-bugstudy.pdf) |
| Dataset2/ | [HAL](https://hal.archives-ouvertes.fr/hal-00769121) | [*CVS-Vintage: A Dataset of 14 CVS Repositories of Java Software*](https://hal.archives-ouvertes.fr/hal-00769121/document) |

**Contributing**: If you like to contribute with a new dataset, drop us a new email.

## Statistics on the competition

| Directory | Total source code files | Lines of code (LOC) |
| --- | --- |--- |
| Dataset1/ | 4421 | 2305784 |
| Dataset2/ | 11141 | 5556153 |

## Command-line interface

To play in the competition, your program takes as input input a folder name, that folder containing input data files (per the format explained above).

```shell
$ your-predictor Files
```

Your programs outputs on the console, for each input data file, the predicted line numbers. Several line numbers can be predicted if the tool is not 100% sure of a single prediction. Warning: by convention, line numbers start from 1 (and not 0).
Your program does not have to predict something for all input files, if there is no clear answer, simply don't output anything, the error computation takes that into account, more information about this in **Loss function** below.

```
<Path1> <line numer>
<Path2> <line numer> <line numer>
<Path3> <line numer>
```

E.g.;
```
/Users/foo/bar/CodRep-competition/Datasets/Dataset1/Tasks/1.txt 42
/Users/foo/bar/CodRep-competition/Datasets/Dataset1/Tasks/2.txt 78 526
...
```

## How to evaluate your competing program

You can evaluate the performance of your program by piping the output to `Baseline/evaluate.py`, for example:
```shell
your-program Files | python evaluate.py
```

The output of `evaluate.py` will be:
```
Total files: 15562
Average line error: 0.944349890554 (the lower, the better)
Recall@1: 0.00777535021206 (the higher, the better)
Recall@5: 0.0377200873924 (the higher, the better)
```

For evaluating specific datasets, use [-d] or [-datasets=] options and specify paths to datasets. The default behaviour is evaluating on all datasets. The path must be absolute path and multiple paths should be separated by `:`, for example:
```shell
your-program Files | python evaluate.py -d /Users/foo/bar/CodRep-competition/Datasets/Dataset1:/Users/foo/bar/CodRep-competition/Datasets/Dataset1
```

Explanation of the output of `evaluate.py`:
* `Total files`: Number of prediction tasks in datasets
* `Average error`: A measurement of the errors of your prediction, as defined in **Loss function** below. This is the only measure used to win the competition.
* `Recall@k`: The percentage of predictions where the correct answer is in your top k predictions. As such, `Recall@1` is the percentage of perfect predictions. We give the recall because it is easily understandable, however, it is not suitable for the competition itself, because it does not has the right properties (explained in `Loss function` below).

## Loss function

The average error is a loss function, output by `evaluate.py`, it measures how well your program performs on predicting the lines to be replaced. The lower the average line  is, the better are your predictions.

The loss function for one prediction task is `tanh(abs({predicted line}-{correct line}))`. The average error is the loss function over all tasks, as calculated as the average of all individual loss.

This loss function is designed with the following properties in mind:
* There is 0 loss when the prediction is perfect
* There is a bounded and constant loss even when the prediction is far away
* Before the bound, the loss is logarithmic
* A perfect prediction is better, but only a small penalty is given to  almost-perfect ones. (in our context, some code line replacement are indeed insensitive to the exact insertion locations).
* The loss is symmetric, continuous and differentiable (except at 0)

We note that the `recall@k` does not comply with all those properties.

## Base line systems f

We provide 5 dumb systems for illustrating how to parse the data and having a baseline performance. These are:
* `guessFirst.py`: Always predict the first line of the file
* `guessMiddle.py`: Always predict the line in the middle of the file
* `guessLast.py`: Always predict the last line of the file
* `randomGuess.py`: Predict a random line in the file
* `maximumError.py`: Predict the worst case, the farthest line from the correct solution

Thanks to the design of the loss function, `guessFirst.py`, `guessMiddle.py`, `guessLast.py` and `randomGuess.py` have the same order of magnitude of error, the value of `average error` are comparable.
