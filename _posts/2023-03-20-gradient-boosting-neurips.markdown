---
layout: single
classes: narrow
title:  "How gradient boosting almost won a NeurIPS competition"
date:   2023-03-20
categories: datascience
comments: true
---

This is a long overdue summary of our experience from the NeurIPS 2022 [Traffic4cast](https://www.iarai.ac.at/traffic4cast/challenge/) (t4c22) competition.
Together with co-author [Andrei Ilie](https://www.linkedin.com/in/catalin-andrei-ilie-848458104/) we managed to grab a second place in the main track and a (close) 4th place in the extended one. Taking into account that this was my second ML competition since 2017, that Andrei had just recently had a baby and was operating on 3 hours of sleep and that we messed up the competition deadline and lost the last day of work, I'm very happy with the result overall. We spent about two months on the competition, with the last two weeks being an especially hackathon-esque slugfest. 

## Competition setup

![counters](https://github.com/iarai/NeurIPS2022-traffic4cast/raw/main/img/task_overview.svg)

The goal of the competition was the following: given a snapshot of the state of a road network, we would like to predict travel times (proxied by a red/yellow/green congestion class label here) on each road (edge) for 15 minutes into the future. This has practical applications for vehicle routing, which is essentially just shortest path finding through a weighted graph, where the weights can be exactly our predicted travel times. 

However, an important crux was that the car counter input data (node) was sparse. This makes sense: unless you are Google, detailed information on all the cars currently on the road network is not available. Instead, there may be some fixed number of vehicle counters at intersections, as happened to be the case in London, Melbourne and Madrid.

As a spatiotemporal edge prediction task based on sparse vertex data, it has also applicability towards research on the spread of COVID and malicious software, or the temporal dynamics of crypto-currency networks. [[1]](https://arxiv.org/pdf/2303.07758.pdf)


## Gradient boosting is not only good for tabular data

NeurIPS is first and foremost a deep learning conference. Also, since the road network is naturally represented as a graph, IARAI presented this as a graph neural network challenge, with the [example](https://github.com/iarai/NeurIPS2022-traffic4cast/tree/main/exploration) notebooks providing various torch dataloaders and GNN utilities. Since I am no GNN expert, and the dummy example notebook would have taken 2 days to train for an epoch on CPU and seemed to crash my SageMaker notebook's VRAM with a tiny batch size, I started looking into other, more familiar solutions. My thinking was that if our approach is significantly worse than the competition, we learn something new and if it's better, then that's also great, so it's a guaranteed win 😀.

Despite how hard some deep learning aficionados wish for it not to be the case, it's common knowledge in data science that gradient boosting methods win competitions on tabular data. A 2022 [survey](https://mlcontests.com/tabular-data/) confirms this is still the case. There are two reasons for this: firstly, they can fit on a complex mix of feature types (categorical, ordinal, numeric) with good accuracy with zero fine-tuning and secondly, depending on your DL architecture, they can be 100-1000x faster to train. 

Most [tabular DL papers](https://sebastianraschka.com/blog/2022/deep-learning-for-tabular-data.html) only account for the first point and try to engineer architectures or architecture ensembles (I saw a paper which trained 10x neural nets with random seeds and averaged their predictions, justifying it with that GBM is also an ensemble of models 🤯). However, being 100x faster is beneficial for ML competitions and real life alike, as increased iteration speed lets you test more hypotheses and improve the model more.

In addition to tabular data, competitions such as [M5](https://www.sciencedirect.com/science/article/pii/S0169207021001874) show that LGB can be the best solution even for large scale multivariate problems. We think that our approach and the scoreboard show that traffic forecasting is "close enough" to a tabular problem that it can also be competitively solved with gradient boosting. More generally, the takeaway seems to be that even if you have _structured data_, if there exists a feasible feature engineering transformation into tabular format, gradient boosting might still be the most practical algorithm. The value proposition of deep learning is of course the reduced need for feature engineering, this is however no free lunch, due to the immensely increased computation cost. So choose your models wisely.


## Our solution

Intuitively there are a few different data sources for predicting the speed on a road segment in the future:

* speed on the same segment in the past
* overall "state" of traffic/speeds in the city currently
* "state" of traffic/speeds in the near vicinity of road of interest
* position of the road relative to others
* static attributes of each road, e.g. number of lanes, pavement type etc

The current state of traffic is provided by the car counter data. If we view this as a regular multivariate time series, we get a $k \times t$ matrix $C$ where $k$ is the number of counters and $t$ the number of observations. The main innovation we deployed was reducing this matrix to the first $p$ principal components and using those as time series features directly. Intuitively, this provides a compressed form of "traffic". We confirm that this is the case by visualising the first two principal components grouped by time in week - they are clearly separable. Note that unlike the winning team, we did not use any external lookup approaches to retrieve exact temporal features, so the principal components are the only proxy for time that our model has.

![pca](https://images-for-web-s3.s3.eu-central-1.amazonaws.com/pca.PNG)

To encode current "nearby" traffic, we tried a somewhat heuristic approach: weighting the counter matrix $C$ with a row-normalised symmetric weight matrix. More details can be found in our [paper](https://arxiv.org/pdf/2211.00157.pdf) but intuitively, this generates a feature which is the distance-weighted mean of nearby counters. This can also be thought of as a hardcoded proxy for a graph convolution layer.

To encode historical traffic for roads, one can use embeddings for neural networks or [target encoding](https://maxhalford.github.io/blog/target-encoding/) for gradient boosting. If your entities (roads) have a lot of density on them, these methods are roughly equivalent, with sparse data target encodings should do slightly better, but they are also dangerously leak-prone.

The best thing about gradient boosting is that you can throw in lat/lng coordinates without any preprocessing. So that was how we encoded road position (note that in OpenStreetMap, road segments are quite short, so there's not much loss of precision from just using a fixed point per line). Finally, road attributes are simply tabular features which are easy to handle with any architecture.

We used LightGBM as our library of choice, because we're familiar with it and experiments with Catboost or XGBoost seemed to take 5-10x longer for standard hyperparameters. We employ two tricks for optimising LGB performance: by using the _init_score_ argument with target encoded features, we significantly reduce training time, as the model can start from a much more accurate base weak learner. We increase the complexity hyperparameter _num_leaves_ to unintuitively high levels like 5k or 10k, in practice on our large dataset this seems to converge faster but not hurt holdout accuracy.

Computationally, our solution for extended track could be run on a 32GB Macbook, and for the core track training models end to end on a larger server took just a few hours. The trained models take <100MB on disk. The code can be found on [GitHub](https://github.com/skandium/t4c22) and the paper on [arXiv](https://arxiv.org/abs/2211.00157).

## Results

Let's have a look at the top final results of core track (with about 10x more data than extended, so arguably harder for GBM approaches).

| **Team**     | **Approach** | **Library** | **Secret Sauce**                           | **Score** |
|--------------|--------------|-------------|--------------------------------------------|-----------|
| ustc-gobbler | GNN          | torch       | GATv2Conv + external time lookup           | 0.8431    |
| Bolt         | GBM          | LGB         | PCA + target encoding                      | 0.8497    |
| oahciy       | GBM          | XGB/LGB     | XGBoost/LightGBM ensemble, target encoding | 0.8504    |
| GongLab      | GNN          | torch       | target encoding                            | 0.8560    |
| TSE          | GBM          | LGB         | k-NN target encoding                       | 0.8736    |
| discovery    | GNN          | torch       | hierarchical graph representation          | 0.8759    |
| ywei         | GNN          | torch       | MLP-based GNN                              | 0.8779    |

First, note how close small are the differences between the top 4 teams. From a practical perspective, the models are all basically the same. Therefore, in a real world scenario the simplicity of the solution would also be important. While I think our PCA-based compression approach is quite nice, I have to acknowledge that the winning team's GNN approach is also very elegant, as they do not do much feature engineering, rather letting the [GATv2Conv](https://arxiv.org/pdf/2105.14491.pdf) layer learn relevant correlations. However, they also reverse engineer the exact time features, so it is unclear whether the GNN would still be more accurate without it. 

Looking at the other teams, it seems that generally GBM does better than GNN. The extended track also sees 3 teams from top 4 using GBM. I believe over the long run we might see GNN models achieving the SotAs on (fixed) graph-based traffic benchmarks, including being able to generalise to unseen graphs i.e. perform _inductive_ graph learning, being able to pool training data from multiple different city graphs etc. But this competition showcases that under a limited time budget, given a new ML prediction task, GBMs can be competitive even on nontabular datasets. It might be that this limited scenario reflects real life projects better.


## Machine learning competitions reflect real life poorly

I often see Kaggle as a bonus requirement for DS/ML roles on LinkedIn. While some exposure can be beneficial, e.g. understanding what are the best ML models for different problems, there is almost no real life benefit from getting to Grandmaster level. 

Experience from t4c22 corroborates this. For the first two weeks we were exploring the data, writing utility functions for "wrangling" it and training some dummy models to see whether there is any signal at all. This is very similar to how new ML projects happen in the industry. Once confident that your model is doing at least something useful, you try to come up with better feature engineering and tune the parameters, this is equivalent to a "v2" model in a real life project.

But everything that happens after that has no resemblance to real problems. For the last month of the competition, we were mostly doing two things: 
1. Trying to engineer target-encoded features based on historical signals, but in a way to minimize leakage.
2. Overfitting to test set by brute forcing solutions with increasing model complexity.

While 2. was mostly an artifact of the competition design - there was no second hidden test set, so all teams spent too much time gaming the leaderboard - the first is something that you'd never do in industry. That's because in real life, you're maximizing a different metric than just accuracy on test set. Your solution should be robust, readable, maintainable and have good latency. So you almost never see ensembling multiple models, stacking or leaky feature engineering, as they would simply generate too much risk.

As wise Pareto foresaw, 95% of the time spent in ML competitions goes into optimizing the last 5% of accuracy, taking on increasing technical debt and complexity. In real life, due to alternative costs you'd surely switch to a new more worthwhile task before that. 

![stacking](http://i.imgur.com/QBuDOjs.jpg)
_Model stacking: Ensembling technique where ML model predictions made on K holdout folds are used as features for a final model_

If you're curious, here is what we did for 1: In addition to the historical travel time on road segment (classic target encoding), it's good to take into account also what was the travel time, _conditional_ on the city's overall traffic. You can think of it as categorising overall vehicle counters into two $state$ buckets: $["low", "high"]$ and then calculating a separate target-encoded feature for each. Since we want to predict the congestion class, we'll get features such as e.g. $proba_{e, green, low} = \mathbb{E} [ \frac{count_{e, green, low}}{count_{e, total, low}}]$ for each edge $e$, where the expectation is taken over training set.


But there is a risk of leakage here. Since the label data was also sparse (in the next 15 minutes, you will only observe true values for edges that saw cars on them), it might be that we have edges with very few samples in the historical data, and any target encoding starts leaking signal. To combat that, for sparse edges we replace $proba_{e, class, state}$ with the median probabilities taken over _all_ sparse edges, so that a high capacity model such a LGB cannot infer the signal directly.

For the extended track where the data was predicting supersegments, we had no sparsity problem, so of course, there's no reason to stop at just 2 traffic state categories. We experimented with 10, 30, even 50, and the accuracy kept increasing!  But this represents exactly the kind of try-hard crazy feature engineering that you wouldn't see in most real life projects!

[1] [Traffic4cast at NeurIPS 2022](https://arxiv.org/pdf/2303.07758.pdf), Neun et al., 2023
