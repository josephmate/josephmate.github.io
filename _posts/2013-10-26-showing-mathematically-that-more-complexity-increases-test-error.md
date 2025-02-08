---
layout: post
title: Showing Mathematically That More Complexity Increases Test Error
date: 2013-10-26 22:25
author: matejoseph
comments: true
categories: [Machine Learning]
---
People who understand machine learning know from experience that having to complex a model causes poor performance when you apply your models to the real world because it does not generalize. However, where does the mathematical intuition for this come from? Up until I saw today's lecture, I only had a sense of this issue because of all the experiments I ran. Today our professor Dr. Ali Ghodsi presented a mathematical proof for a radial basis function network decreasing in test performance as a result of having too many hidden nodes. In the follow post I give the necessary background on RBFs and a summary of the proof for overfitting due to too many hidden nodes.
<h2>Background - What is a Radial Basis Function Network (RBF)?</h2>
It's easy to imagine RBF as a neural network with a few differences.
<ol>
	<li>There is only one hidden layer.</li>
	<li>There are only weights going from the input nodes to the hidden layer.</li>
	<li>The activation function applied on the hidden layer is the normal distribution.</li>
	<li>You cluster your data and take their mean and variance to determine the parameters for the normal distributions used in the activation function above.</li>
</ol>
<h2>Result</h2>
Details of the proof are given in Professor Ghodsi's and Dale Schuurmans's paper: <a title="Automatic Basis Selection Techniques for RBF Networks" href="http://www.math.uwaterloo.ca/~aghodsib/papers/journalofnn.pdf" target="_blank">Ali Ghodsi, Dale Schuurmans, Automatic basis selection techniques for RBF networks, Neural Networks, Volume 16, Issues 5–6, June–July 2003, Pages 809–816</a>.

By assuming that the noise in the training and test data sets are guassian, the paper proves:
$latex \Large Error(M) = error(M) - N\sigma^2 + 2\sigma(M+1) $
Where
M is the number of hidden nodes
N is the number of data points in the training set
σ is the variance of the gaussian noise assumed to be in the data set
Error(M) is the Test error as a function of the number of hidden units
error(M) is the training error as a function of the number of hidden units
<h2>Conclusion</h2>
Hopefully now you have some mathematically intuition for why having too complex a model causes poor performance on new examples. Unfortunately, the derivation in the paper does not extend beyond RBF networks. However, the tools used in the proof might be applicable to other machine learning algorithms.

