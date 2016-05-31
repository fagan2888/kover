---
title: Machine Learning Module
tags:
keywords: learning, engine, set covering machine, SCM, kover, genomics, k-mer, machine learning
last_updated: May 30, 2016
summary: "Overview of the machine learning functionality"
---

## Learning models

This command is used to learn a model from a Kover dataset. It provides an interface on the Set Covering Machine algorithm.

```
usage: kover learn [-h] --dataset DATASET --split SPLIT --model-type
                   {conjunction,disjunction} [{conjunction,disjunction} ...]
                   --p P [P ...] --max-rules MAX_RULES
                   [--max-equiv-rules MAX_EQUIV_RULES]
                   [--hp-choice {bound,cv,none}] [--bound-delta BOUND_DELTA]
                   [--bound-max-genome-size BOUND_MAX_GENOME_SIZE]
                   [--random-seed RANDOM_SEED] [--n-cpu N_CPU]
                   [--output-dir OUTPUT_DIR] [-x] [-v]

Learn a model from data

optional arguments:
  -h, --help            show this help message and exit
  --dataset DATASET     The Kover dataset from which to learn.
  --split SPLIT         The identifier of the split of the dataset to use for
                        learning.
  --model-type {conjunction,disjunction} [{conjunction,disjunction} ...]
                        Hyperparameter: The type of model to learn,
                        conjunction (logical-AND) or disjunction (logical-OR).
                        You can specify multiple space separated values.
  --p P [P ...]         Hyperparameter: The value of the trade-off parameter
                        in the rule scoring criterion. You can specify
                        multiple space separated values.
  --max-rules MAX_RULES
                        The maximum number of rules that can be included in
                        the model.
  --max-equiv-rules MAX_EQUIV_RULES
                        The maximum number of equivalent rules to report for
                        each rule in the model. This only affects model
                        interpretation. Use the default unless you expect that
                        the rules in the model will be equivalent to more than
                        10000 other rules.
  --hp-choice {bound,cv,none}
                        The strategy used to select the best values for
                        hyperparameters. The default is k-fold cross-
                        validation, where k is the number of folds defined in
                        the split. Other strategies, such as bound selection
                        are available. Using none selects the first value
                        specified for each hyperparameter.
  --bound-delta BOUND_DELTA
                        The probabilistic bound on the error rate will be
                        valid with probability 1-delta. The default value is
                        0.05.
  --bound-max-genome-size BOUND_MAX_GENOME_SIZE
                        The maximum size, in base pairs, of anygenome in the
                        dataset. If you are unsure about this value, you
                        should use an over-estimate. This will only affect the
                        tightness of the bound on the error rate. By default
                        number of k-mers in the dataset is used.
  --random-seed RANDOM_SEED
                        The random seed used for any random operation. Set
                        this if only if you require that the same random
                        choices are made between repeats.
  --n-cpu N_CPU         The number of CPUs used to select the hyperparameter
                        values. Make sure your computer has enough RAM to
                        handle multiple simultaneous trainings of the
                        algorithm and that your storage device will not be a
                        bottleneck (simultaneous reading).
  --output-dir OUTPUT_DIR
                        The directory in which to store Kover's output. It
                        will be created if it does not exist.
  -x, --progress        Shows a progress bar for the execution.
  -v, --verbose         Sets the verbosity level.
```

### Understanding the hyperparameters

A hyperparameter is a free parameter of a learning algorithm that must be set by the user, prior to seeing the data.
Generally, the user defines a set of candidate values for each hyperparameter and uses a strategy, such as
[cross-validation](https://en.wikipedia.org/wiki/Cross-validation_(statistics)), to select the best value.

Kover has 2 hyperparameters:

* The model type (option --model-type)
* The trade-off parameter (option --p)

#### The model type

Kover can learn models that are conjunctions (logical-AND) or disjunctions (logical-OR) of rules (presence or absence of k-mers). The optimal model type
depends on the phenotype and, in absence of prior knowledge, both types should be tried.

This is achieved by using:
```
--model-type conjunction disjunction
```

#### The trade-off parameter (p)

Kover selects the rules to include in the model based on a scoring function that is applied to each rule. This function
consists of a trade-off between the number of errors incurred on each class of examples (i.e., phenotype = 0 and phenotype = 1).

For example, when learning a conjunction, the score of each rule is given by:

$$ S_i = N_i - p \times \overline{P_i}$$

where:

* $S_i$ is the score of rule i
* $N_i$ is the number of negative (phenotype=0) examples that are correctly classified by the rule
* $\overline{P_i}$ is the number of positive (phenotype = 1) examples that are incorrectly classified by the rule

Hence, larger values of p increase the importance of errors made on negative examples, while smaller values will increase
the importance of errors made on the positive examples.

Again, the optimal value is problem specific and many values must be tried. For example, we often use:

```
--p 0.1 0.178 0.316 0.562 1.0 1.778 3.162 5.623 10.0 999999.0
```

### Hyperparameter selection strategies

The are many strategies to select the values of hyperparameters. Kover implements two of them, which are described below:

* k-fold cross-validation
* Risk bound selection

#### k-fold cross-validation

This is the most widely used method in machine learning. The data is partitioned into a training and a testing set.
The training set is then used to determine the best hyperparameter values and to learn the final model. The testing set
is set aside and subsequently used to evaluate the accuracy of the final model.

In k-fold cross-validation, the training data is partitioned into k blocks of equal size (called folds). Then, for each
possible combination of values for the hyperparameters, the following procedure is applied. One at a time, the folds are
set aside for testing and the remaining (k-1) folds are used to train the algorithm. The score of a combination of values
is measured as the average number of correct predictions on the left out folds.

Below is an illustration of 5-fold cross-validation:

<img src='images/cross_validation.png' title="5-fold cross-validation" style="width: 50%; height: 50%" />


The combination of hyperparameter values with the greatest score is then selected and used to retrain
the learning algorithm from the entire training set, yielding the final model.

To use this strategy in Kover, you must:

- Define the number of folds when using the ``` kover dataset split``` command.
- Use ```--hp-choice cv```

<br/>
**Choosing the number of folds**

The general standard is 5 or 10 fold cross-validation. For smaller datasets, the number of folds can be increased,
which results in more data available for training the algorithm and thus, a better estimate of the score of the
hyperparameter values.

**Note**

The major disadvantage of this method is that it requires training the algorithm k+1 times for each combination of hyperparameter
values. This can be computationally intensive for larger datasets. If you encounter long running times with this method,
we recommend trying risk bound selection, which is described below.


#### Risk bound selection

Risk bound selection is a great alternative to cross-validation for larger datasets. It is much faster and requires a
single training of the learning algorithm to score each combination of hyperparameter values. This method trains the
algorithm on the entire training set using each hyperparameter combination and uses a mathematical expression to estimate
an upper bound on the error rate of the obtained model. The combination of values that yields the smallest bound value is
retained.

**Note**
While this method is fast, the bound value can be inaccurate if there are too few learning examples. The minimum number
of examples is problem dependent.