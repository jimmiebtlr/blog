## ML Experiment Log - Polynomial Network + MNIST

I wanted to try implementing previous work [nn + polynomial solvers](https://www.jimmiebtlr.com/ml-experiment-log-nn-polynomial-solvers-pt-2) against a larger problem, in this case MNIST.

For anyone hopping in here, I'm exploring adapting neural networks to use polynomial solvers instead of gradient descent.  I'm looking specifically at symbolic math for this currently, though may look at numerical solving methods in the future.  Performance wise, execution time is likely factorial of number of variables being solved at once (not certain though).  Currently I've been working with solving a single trainable variable at a time.

## What

This is an attempt to apply the previous attempt of using polynomial solvers against the MNIST dataset and image classification.

## Implementation

I implemented a simple network shaped as 

conv -> linear layer -> 0-9 outputs

With a over 6k trainable variables it was a bit of a stress test, with I think decent results.  I didn't add any non-linearity via activation layer or multiplication though, and the network is extremely simple.

The full code can be found [here](https://colab.research.google.com/gist/jimmiebtlr/1fcd710fb298525d0a772bc369502e7b/polynomialnetworkmnist.ipynb).

### Important notes

TrainX is limited to first 100 training data points to speed things up, note this probably makes things harder on the accuracy side.  I was hoping the training loss function calculation would be O(1), but it seems to slow down as the loss function as built.  I thought it would be constant time since I think the terms would be able to be combined after expansion to the loss function the same size as more data is added.  On a related note, calling `expand` on the loss function causes OOM.  


## Results

~44% accuracy after training for around an hour (cpu only, not even 1 train per trainable varaible).

I'm not sure if we're into overfitting territory or under fitting, the network is quite simple but it's still learning at about the same rate as it started, so I think we're still under trained.

Quite slow at the moment, but there are many opportunities for optimization.  