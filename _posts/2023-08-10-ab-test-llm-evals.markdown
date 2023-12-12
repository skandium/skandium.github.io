---
layout: single
classes: wide
title:  "Why you should A/B test your LLM evals"
date:   2023-08-10
categories: technical
comments: true
---

## TL;DR

LLM stochasticity poses a problem for evaluating whether new commits improve a project's performance or not - the solution is to run a paired difference test on repeated measures of the same eval set for maximal statistical power.

## Introduction

OpenAI's GPT-4 outputs are not deterministic. [[1]](https://twitter.com/yuntiandeng/status/1641108596510343168)[[2]](https://community.openai.com/t/why-is-gpt-4-giving-different-answers-with-same-prompt-temperature-0/143513) Leaving aside [anecdotal](https://arstechnica.com/information-technology/2023/07/is-chatgpt-getting-worse-over-time-study-claims-yes-but-others-arent-sure/) performance changes over long time horizons, simply sending the same API request 10 times in a row can give you 10 different answers. Working with GPT-4 at Bolt, we noticed that these differences are not just mere paraphrasing - it could also be that an [AutoGPT](https://github.com/Significant-Gravitas/Auto-GPT)-like agent chooses an entirely different set of actions across multiple requests. It looks to be caused by the nature of how log probabilities are stored and used in [parallel GPU calculations](https://community.openai.com/t/a-question-on-determinism/8185/2). Thus, as a fundamental problem, it might affect all modern instruction tuned AGI models.

However, as most practitioners also [know](https://eugeneyan.com/writing/llm-patterns/#retrieval-augmented-generation-to-add-knowledge), LLM development needs to be evals-driven (EDD) - that means every prompt, code or model modification should run against some fixed set of task-specific test cases that have a quantified success measure to either greenlight or block new commits. For simplification, assume we have a simple binary accuracy measure. This is fairly realistic if your LLM is deployed as an agent and needs to choose from a discrete set of actions / API calls to perform.

Let's say by anecdotal inspection you discover a systematic issue in the LLM outputs and want to fix it with improved prompt engineering. As prompts tend to be not the most modular things, it's hard to know if these changes do not hurt overall accuracy. Therefore, we need to measure both the old (A) and new (B) branches on our evals set  and compare the accuracies. Alright, we did that - the values are 73% and 76% respectively. Great! Does that mean we are safe to roll it out? This brings us back to the original problem of stochasticity. On our evals set on Bolt we saw that on a sample of 64 cases, the accuracy could vary in consequent A/A eval runs from 65% to 75%! This renders a naive comparison of accuracies essentially useless, as unless the changes have a dramatically large impact, they can get overshadowed by background variation. 

Luckily estimating unknown population parameters from a noisy sample is the fundamental problem of statistics, so let's see if developers can use a few tricks from the field.

## Statistics to the rescue

More stats-savvy readers lament: why not just increase the sample size? It's true that if we increase it to infinity, by the [law of large numbers](https://en.wikipedia.org/wiki/Law_of_large_numbers) we will converge to the true accuracy ratio in either branch. Alas, collecting curated validation samples can be costly, and starting from thousands can also become slow to evaluate and cost-prohibitive for SotA models such as GPT-4.

However, even with a limited sample size, we can use statistical hypothesis testing between the samples from the two branches. For a binary conversion metric such as accuracy, there are multiple suitable tests, but below I use the Welch t-test and the chi-square contingency table test.

We can simulate the scenario described above in Python:

```python
import numpy as np  
from scipy.stats import truncnorm, ttest_ind, chi2_contingency  
import pandas as pd
from tqdm import tqdm

np.random.seed(1337)

TRUE_EFFECT_SIZE = 0.05  # This is the percentage point effect size treatment gives
N_TEST_CASES = 100
  
## First, simulate "true" values of accuracy for the test cases.
# This is the probability that a given LLM call will give the correct answer -
# and directly related to the difficulty of the test case.
# This distribution is a parameter of the simulation and could be changed
# for uniform, truncated normal etc.
values = [0.15, 0.5, 0.9]  
weights = [0.25, 0.15, 0.6]  
weights = np.array(weights) / sum(weights)  
probabilities_control = np.random.choice(values, size=N_TEST_CASES, p=weights)  
probabilities_treatment = np.clip(probabilities_control + TRUE_EFFECT_SIZE, 0, 1)

## Second, simulate the model's stochastic behaviour  
def llm_call(probabilities):  
	return np.array([np.random.choice([0, 1], p=[1 - p, p]) for p in probabilities])

## Finally, we can simulate the results of LLM eval calls

MC_TRIALS = 500
# Arrays to hold p-values  
p_values_t_test = np.zeros(MC_TRIALS)  
p_values_chi_sq = np.zeros(MC_TRIALS)  
  
# Monte Carlo trials  
for i in tqdm(range(MC_TRIALS)):  	 
	results_control = llm_call(probabilities_control)  
	results_treatment = llm_call(probabilities_treatment)  
	  
	data = pd.DataFrame({  
	'result': np.concatenate([results_control, results_treatment]),  
	'treatment': np.concatenate([np.zeros(N_TEST_CASES), np.ones(N_TEST_CASES)]),  
	'test_case': np.concatenate([np.arange(N_TEST_CASES), np.arange(N_TEST_CASES)])  
	})  
		  	  
	# Perform a t-test  
	t_test_result = ttest_ind(data[data['treatment'] == 0]['result'],  
	data[data['treatment'] == 1]['result'])  
	p_values_t_test[i] = t_test_result.pvalue  
	
	# Perform a chi-square test  
	contingency_table = pd.crosstab(data['treatment'], data['result'])  
	_, p, _, _ = chi2_contingency(contingency_table)  
	p_values_chi_sq[i] = p

# Compare power  
power_t_test = np.mean(p_values_t_test < 0.05)  
power_chi_sq = np.mean(p_values_chi_sq < 0.05)

print(f"Power of the t-test for each iteration: {power_t_test}")  
print(f"Power of the chi-squared test for each iteration: {power_chi_sq}")

>>Power of the t-test for each iteration: 0.046
>>Power of the chi-squared test for each iteration: 0.022
```


The results do not look too promising! 😱 For an accuracy baseline of roughly 67% and a true effect size of 5pp, t-test detects a significant effect only 5% of the times, and chi-squared test does even worse at 2%! 

## Paired difference test

Fortunately there is another source of variation we can exploit - namely that the test cases are different. Some are very difficult - and therefore have a lower probability of being correct. Technically, we are dealing with a paired observation here - each test case is measured for both branches. So to achieve better statistical power, we can use a paired t-test instead. This is as simple as adding the following code:

```python
from scipy.stats import ttest_rel
...
p_values_paired_t_test = np.zeros(MC_TRIALS)
...
	# Perform a paired t-test  
	paired_t_test_result = ttest_rel(data[data['treatment'] == 0]['result'],  
	data[data['treatment'] == 1]['result'])  
	p_values_paired_t_test[i] = paired_t_test_result.pvalue
...
power_paired_t_test = np.mean(p_values_paired_t_test < 0.05)
print(f"Power of the paired t-test for each iteration: {power_paired_t_test}")

>>Power of the paired t-test for each iteration: 0.154
```
This yields an improvement as expected, but still does not seem too good. 

Paired t-test is usually used for [paired data](https://en.wikipedia.org/wiki/Paired_data) in the special case where we have exactly two sets of measurements for each group (for example before/after measurement in a drug trial). However, why limit ourself to 2? Since the number of LLM evals calls is entirely in our control, we could run the eval for each case an arbitrary $N$ number of times and this should provide us with even more variation to exploit. 

## Fixed/mixed effects

Instead of the paired t-test, with $N$ measurements (where $N$ can actually be different between control and treatment), [StackExchange](https://stats.stackexchange.com/questions/507154/paired-t-test-with-multiple-measurements-per-pair) guided me towards the [mixed effects model](https://www.tjmahr.com/plotting-partial-pooling-in-mixed-effects-models/) to estimate the treatment coefficient. This model seems almost perfectly suited for our use case, as we know there should be some background variation of accuracy depending on the test case specifics, but we don't necessarily want to explicitly estimate a coefficient (fixed effect) for it - we only care about the treatment effect. To implement this in Python, we need to add an additional `N_ITERATIONS` loop within a trial, aggregate trial data over iterations, and add a statsmodels fit. For fun, let's also add some benchmarks - the [fixed effects](https://en.wikipedia.org/wiki/Fixed_effects_model) (dummy variable) model, just naive linear regression and running a paired t-test on data that has been averaged within each group.

```python
# Arrays to hold p-values  
p_values_t_test = np.zeros(MC_TRIALS)  
p_values_chi_sq = np.zeros(MC_TRIALS)  
p_values_paired_t_test = np.zeros(MC_TRIALS)  
p_values_avg_paired_t_test = np.zeros(MC_TRIALS)  
p_values_mixed_effects = np.zeros(MC_TRIALS)  
p_values_fixed_effects = np.zeros(MC_TRIALS)  
p_values_ols = np.zeros(MC_TRIALS)

N_ITERATIONS = 5

for i in tqdm(range(MC_TRIALS)):  
	trial_datas = []  
	for _ in range(N_ITERATIONS):  
		results_control = llm_call(probabilities_control)  
		results_treatment = llm_call(probabilities_treatment)  
  
	# Create a DataFrame for the mixed effects model  
	trial_datas.append(pd.DataFrame({  
	'result': np.concatenate([results_control, results_treatment]),  
	'treatment': np.concatenate([np.zeros(N_TEST_CASES), np.ones(N_TEST_CASES)]),  
	'test_case': np.concatenate([np.arange(N_TEST_CASES), np.arange(N_TEST_CASES)])  
	}))
	data = pd.concat(trial_datas)  
  
	# Fit the mixed effects model  
	model = smf.mixedlm("result ~ treatment", data, groups=data['test_case'])  
	model_fit = model.fit()  
	p_values_mixed_effects[i] = model_fit.pvalues['treatment']  
	  
	# Fit fixed effects model  
	model = smf.ols('result ~ treatment + C(test_case)', data=data)  
	model_fit = model.fit()  
	p_values_fixed_effects[i] = model_fit.pvalues['treatment']  
	  
	# Fit OLS  
	model = smf.ols('result ~ treatment', data=data)  
	model_fit = model.fit()  
	p_values_ols[i] = model_fit.pvalues['treatment']  
	  
	# Perform a paired t-test  
	paired_t_test_result = ttest_rel(data[data['treatment'] == 0]['result'],  
	data[data['treatment'] == 1]['result'])  
	p_values_paired_t_test[i] = paired_t_test_result.pvalue  
	  
	# Perform a paired t-test on averaged data  
	data_avg = data.groupby(["test_case", "treatment"])["result"].mean().reset_index()  
	paired_t_test_avg_result = ttest_rel(data_avg[data_avg['treatment'] == 0]['result'],  
	data_avg[data_avg['treatment'] == 1]['result'])  
	p_values_avg_paired_t_test[i] = paired_t_test_avg_result.pvalue  
	  
	# Perform a t-test  
	t_test_result = ttest_ind(data[data['treatment'] == 0]['result'],  
	data[data['treatment'] == 1]['result'])  
	p_values_t_test[i] = t_test_result.pvalue  
	  
	# Perform a chi-square test  
	contingency_table = pd.crosstab(data['treatment'], data['result'])  
	_, p, _, _ = chi2_contingency(contingency_table)  
	p_values_chi_sq[i] = p

# Compare power  
power_t_test = np.mean(p_values_t_test < 0.05)  
power_chi_sq = np.mean(p_values_chi_sq < 0.05)
p_values_ols = np.mean(p_values_ols < 0.05)  
power_paired_t_test = np.mean(p_values_paired_t_test < 0.05)  
power_avg_paired_t_test = np.mean(p_values_avg_paired_t_test < 0.05)  
power_mixed_effects = np.mean(p_values_mixed_effects < 0.05)  
power_fixed_effects = np.mean(p_values_fixed_effects < 0.05)  
  
print(f"Power of the t-test for each iteration: {power_t_test}")  
print(f"Power of the chi-squared test for each iteration: {power_chi_sq}")
print(f"Power of the OLS model for each iteration: {p_values_ols}")
print(f"Power of the paired t-test for each iteration: {power_paired_t_test}")  
print(f"Power of the avg paired t-test for each iteration: {power_avg_paired_t_test}")  
print(f"Power of the mixed effects model for each iteration: {power_mixed_effects}")  
print(f"Power of the fixed effects model for each iteration: {power_fixed_effects}")

>>Power of the t-test for each iteration: 0.368
>>Power of the chi-squared test for each iteration: 0.336
>>Power of the OLS model for each iteration: 0.368
>>Power of the paired t-test for each iteration: 0.592
>>Power of the avg paired t-test for each iteration: 0.59
>>Power of the mixed effects model for each iteration: 0.584
>>Power of the fixed effects model for each iteration: 0.584
```

A couple of points are immediately striking:

* The power of all tests increases when increasing `N_ITERATIONS`. Even if we are not sure about the optimal test setup, scaling repeated measurements seems useful. For a sanity check, the interested reader can verify that power is near zero for all tests if we set `TRUE_EFFECT_SIZE` equal to 0.
* t-test, chi-squared and OLS are clearly worse than the others - this is because they do not leverage the paired nature of the data.
* Paired t-test, averaged paired t-test, mixed effects and fixed effects models have very similar results, with the latter being identical. 

This raises a few interesting questions:

Isn't averaging multiple measurements data before paired comparison bad? Wouldn't you lose useful variation? It turns out that yes - the averaged power is always less than paired one - but the difference is very small, which might partly be an artefact of the simulation setup.

There are research [papers](https://link.springer.com/article/10.1007/s11135-018-0802-x) written about the subtle differences between mixed effects and random effects. How can they be identical? It turns out that if you are only interested in the _treatment effect_ rather than the group effects (here, what is the estimated accuracy rate of each individual test case) and you also have a balanced dataset with no missing values, then a paired t-test, mixed effects and random effects are actually **all** equivalent! Marco Laube has a nice [proof](https://medium.com/@marco_laube/the-paired-t-test-and-linear-mixed-models-185a084d7813) on this. In the appendix, I extend it to our use case with multiple measurements.

To me personally, the equivalence of a simple paired test and fixed effects was not intuitive at first. In the first case, we simply stack repeated measurements on top of each other in *long* format and treat them as independent pairs, while in the latter if we measured the performance on 2 different test cases 1 million times, we would get a nice estimate on the case specific performance. It boils down to what we want to measure - since it's the treatment effect $\hat{\beta_1}$ rather than $\hat{u_2}$, the choice between methods does not matter that much.

Finally, it's worth noting that scaling `N_ITERATIONS` is no magic solution for small evals samples. If we managed to set `N_ITERATIONS=int(1e3)`, our $\hat{\beta_1}$ *will* converge to the population treatment effect value. However, the interpretation is not the one we're interested in necessarily. It will only tell us that a change to the prompts caused an effect of size $\hat{\beta_1}$ *on the limited set of test cases*. This does not necessarily transfer to good generalisation ability to other requests the model might see in production - in the worst case, we've chosen completely irrelevant test cases so the exercise is useless. So it is always a good idea to also scale `N_TEST_CASES` and make sure they correspond to real problems you expect your LLM to solve.


## Appendix

Reusing Marco Laube's notation - if our model is given by 

$$y_{ijk} = \beta_{0} + \beta_{1} \mathbf{1}_{j=2} + u_{i} + \epsilon_{ijk} $$

where $j$ denotes treatment in $\{1,2\}$ and $k$ is the number of repeated measurements we have per each group (below example uses $k=2$ and $n=2$ to exemplify), we can write:

$$
 \begin{bmatrix}
     y_{111} \\
     y_{112} \\ 
     y_{121} \\
     y_{122} \\
     y_{211} \\
     y_{212} \\
     y_{221} \\
     y_{222}  \\
     \end{bmatrix}
     = 
     \begin{bmatrix}
         1 & 0 & 0 \\
         1 & 0 & 0 \\
         1 & 1 & 0 \\
         1 & 1 & 0 \\
         1 & 0 & 1 \\
         1 & 0 & 1 \\
         1 & 1 & 1 \\
         1 & 1 & 1 \\ 
     \end{bmatrix}
      \times
     \begin{bmatrix}
         \beta_0 \\
         \beta_1 \\ 
         u_2 \\ 
     \end{bmatrix}
$$

We then get that $k$ scales the matrix $X^TX$ (makes sense, as we're essentially duplicating data in $X$).

$$
X^{T}X = k
 \begin{bmatrix}
 2n & n & 2 \\  
 n & n & 1 \\
 2 & 1 & 2 \\
 \end{bmatrix}
$$

and the matrix inverse is divided by k: 

$$
(X^{T}X)^{-1} = \frac{1}{k}
 \begin{bmatrix}
 \frac{n+1}{2n} & -\frac{1}{n} & -\frac{1}{2}  \\[6pt] 
 -\frac{1}{n} & \frac{2}{n} & 0  \\[6pt]
 -\frac{1}{2} & 0 & 1 \\[6pt]
 \end{bmatrix}
$$

Therefore

$$
(X^{T}X)^{-1} X^T \in \mathbf{R}^{3\times 8} = \frac{1}{2}
 \begin{bmatrix}
 \frac{n+1}{2n} & \frac{n+1}{2n} & \frac{n-1}{2n} & \frac{n-1}{2n} & \frac{1}{2n} & \frac{1}{2n} & -\frac{1}{2n} & -\frac{1}{2n}  \\[6pt]  
 -\frac{1}{n} & -\frac{1}{n} & \frac{1}{n} & \frac{1}{n} & -\frac{1}{n} & -\frac{1}{n} & \frac{1}{n} & \frac{1}{n} & \\[6pt]
 -\frac{1}{2} & -\frac{1}{2} & -\frac{1}{2} & -\frac{1}{2} & \frac{1}{2} & \frac{1}{2} & \frac{1}{2} & \frac{1}{2}  \\[6pt]
 \end{bmatrix}
$$

Note that we will have $k$ duplicate columns - in this case 2.
The OLS estimator is given by:

$$
\hat{\beta} =  (X^{T}X)^{-1} X^Ty =  
\frac{1}{k}
\begin{bmatrix}
\frac{n+1}{2n}(y_{111} + y_{112}) + \frac{n-1}{2n}(y_{121} + y_{122}) + \frac{1}{2n}(y_{211}+y_{212}) - \frac{1}{2n}(y_{221} + y_{222}) \\
\frac{1}{n} \big( (y_{121} + y_{122}) - (y_{111} + y_{112}) + (y_{221}+y_{222}) - (y_{211}+y_{212}) \big) \\[6pt]
\frac{1}{2} \big( (y_{211}+y_{212}+y_{221}+y_{222}) - (y_{111} + y_{112} + y_{121} + y_{122})
\end{bmatrix}
$$

We see that
$$ \hat{\beta_1} = \frac{1}{n} \sum^n \frac{1}{k} \big( \sum^k y_{i2k} - y_{i1k} \big)  $$

Let's denote $$ \gamma_{ij} = \sum^k y_{ijk}$$ then we have $$ \hat{\beta_1} = \frac{1}{n} \sum^n \frac{1}{k} \big( \gamma_{i2} - \gamma_{i1} \big)$$

We see that the treatment effect estimator simply becomes the average of the average (over $k$) differences of the independent variable. The rest of the proof should follow Marco Laube's [derivations](https://medium.com/@marco_laube/the-paired-t-test-and-linear-mixed-models-185a084d7813). I barely trust my own math though, so here's a sanity check based on our simulation data from above: if the estimators are equivalent, we should:

* see a near 1 correlation in their p-values.
* see a very similar distribution of their p-values.
 
 Let's check the full correlation matrix of all estimators and the corresponding medians:

|                   | t-test | chi-sq | OLS    | paired t-test | paired avg t-test | mixed effects | fixed effects |
|-------------------|--------|--------|--------|---------------|-------------------|---------------|---------------|
| t-test            | 1      | 0.9996 | 1      | 0.9746        | 0.9721            | 0.9748        | 0.9749        |
| chi-sq            |        | 1      | 0.9996 | 0.9686        | 0.9663            | 0.9687        | 0.9689        |
| OLS               |        |        | 1      | 0.9746        | 0.9721            | 0.9748        | 0.9749        |
| paired t-test     |        |        |        | 1             | 0.9956            | 0.9995        | 0.9995        |
| paired avg t-test |        |        |        |               | 1                 | 0.9954        | 0.9954        |
| mixed effects     |        |        |        |               |                   | 1             | 1             |
| fixed effects     |        |        |        |               |                   |               | 1             |

We see that:

1. t-test is very similar to chi-squared, and exactly equivalent to OLS. The latter is a fairly well known result from statistics. 
2. Indeed, the correlation between paired t-test, mixed effects and fixed effects is near 1. 

Since in hypothesis testing we also care about the absolute levels rather than only correlations, we can also look at the median values for an estimator's p-value distribution:

| Estimator         | Median p-value |
|-------------------|----------------|
| t-test            | 0.0959         |
| chi-sq            | 0.1104         |
| OLS               | 0.0959         |
| paired t-test     | 0.0266         |
| paired avg t-test | 0.031          |
| mixed effects     | 0.0247         |
| fixed effects     | 0.025          |

We see that:

1. Even though highly correlated, chi-squared test is worse than t-test in this case.
2. Even though paired t-test was slightly more sensitive when we were looking at the 5% threshold, based on the medians, mixed and fixed effects are better. However, the differences are very small, small enough to put down to implementation differences between the libraries, so we can indeed say that the methods are equivalent 🙂 Since paired t-test is easier to do, I'd recommend this approach.



