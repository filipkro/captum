![Captum Logo](./website/static/img/captum_logo.png)

<hr/>

[![Conda](https://img.shields.io/conda/v/pytorch/captum.svg)](https://anaconda.org/pytorch/captum)
[![PyPI](https://img.shields.io/pypi/v/captum.svg)](https://pypi.org/project/captum)
[![CircleCI](https://circleci.com/gh/pytorch/captum.svg?style=shield)](https://circleci.com/gh/pytorch/captum)

Captum is a model interpretability and understanding library for PyTorch.
Captum means comprehension in latin and contains general purpose implementations
of integrated gradients, saliency maps, smoothgrad, vargrad and others for
PyTorch models. It has quick integration for models built with domain-specific
libraries such as torchvision, torchtext, and others.

*Captum is currently in beta and under active development!*


#### About Captum

With the increase in model complexity and the resulting lack of transparency, model interpretability methods have become increasingly important. Model understanding is both an active area of research as well as an area of focus for practical applications across industries using machine learning. Captum provides state-of-the-art algorithms, including Integrated Gradients, to provide researchers and developers with an easy way to understand which features are contributing to a model’s output.

For model developers, Captum can be used to improve and troubleshoot models by facilitating the identification of different features that contribute to a model’s output in order to design better models and troubleshoot unexpected model outputs.

Captum helps ML researchers more easily implement interpretability algorithms that can interact with PyTorch models. Captum also allows researchers to quickly benchmark their work against other existing algorithms available in the library.

#### Target Audience

The primary audiences for Captum are model developers who are looking to improve their models and understand which features are important and interpretability researchers focused on identifying algorithms that can better interpret many types of models.

Captum can also be used by application engineers who are using trained models in production. Captum provides easier troubleshooting through improved model interpretability, and the potential for delivering better explanations to end users on why they’re seeing a specific piece of content, such as a movie recommendation.

## Installation

**Installation Requirements**
- Python >= 3.6
- PyTorch >= 1.2


##### Installing the latest release

The latest release of Captum is easily installed either via
[Anaconda](https://www.anaconda.com/distribution/#download-section) (recommended):
```bash
conda install captum -c pytorch
```
or via `pip`:
```bash
pip install captum
```


##### Installing from latest master

If you'd like to try our bleeding edge features (and don't mind potentially
running into the occasional bug here or there), you can install the latest
master directly from GitHub:
```bash
pip install git+https://github.com/pytorch/captum.git
```

**Manual / Dev install**

Alternatively, you can do a manual install. For a basic install, run:
```bash
git clone https://github.com/pytorch/captum.git
cd captum
pip install -e .
```

To customize the installation, you can also run the following variants of the
above:
* `pip install -e .[insights]`: Also installs all packages necessary for running Captum Insights.
* `pip install -e .[dev]`: Also installs all tools necessary for development
  (testing, linting, docs building; see [Contributing](#contributing) below).
* `pip install -e .[tutorials]`: Also installs all packages necessary for running the tutorial notebooks.

To execute unit tests from a manual install, run:
```bash
# running a single unit test
python -m unittest -v tests.attr.test_saliency
# running all unit tests
pytest -ra
```

## Getting Started
Captum helps you interpret and understand predictions of PyTorch models by
exploring features that contribute to a prediction the model makes.
It also helps understand which neurons and layers are important for
model predictions.

Currently, the library uses gradient-based interpretability algorithms
and attributes contributions to each input of the model with respect to
different neurons and layers, both intermediate and final.

Let's apply some of those algorithms to a toy model we have created for
demonstration purposes.
For simplicity, we will use the following architecture, but users are welcome
to use any PyTorch model of their choice.


```
import numpy as np

import torch
import torch.nn as nn

from captum.attr import (
    GradientShap,
    DeepLift,
    DeepLiftShap,
    IntegratedGradients,
    LayerConductance,
    NeuronConductance,
    NoiseTunnel,
)

class ToyModel(nn.Module):
    def __init__(self):
        super().__init__()
        self.lin1 = nn.Linear(3, 3)
        self.relu = nn.ReLU()
        self.lin2 = nn.Linear(3, 2)
        self.sigmoid = nn.Sigmoid()

        # initialize weights and biases
        self.lin1.weight = nn.Parameter(torch.arange(0.0, 9.0).view(3, 3))
        self.lin1.bias = nn.Parameter(torch.zeros(1,3))
        self.lin2.weight = nn.Parameter(torch.arange(0.0, 6.0).view(2, 3))
        self.lin2.bias = nn.Parameter(torch.ones(1,2))

    def forward(self, input):
        return self.sigmoid(self.lin2(self.relu(self.lin1(input))))
```

Let's create an instance of our model and set it to eval mode.
```
model = ToyModel()
model.eval()
```

Next, we need to define simple input and baseline tensors.
Baselines belong to the input space and often carry no predictive signal.
Zero tensor can serve as a baseline for many tasks.
Some interpretability algorithms such as Integrated
Gradients, Deeplift and GradientShap are designed to attribute the change
between the input and baseline to a predictive class or a value that the neural
network outputs.

We will apply model interpretability algorithms on the network
mentioned above in order to understand the importance of individual
neurons/layers and the parts of the input that play an important role in the
final prediction.

To make computations deterministic, let's fix random seeds.

```
torch.manual_seed(123)
np.random.seed(123)
```

Let's define our input and baseline tensors. Baselines are used in some
interpretability algorithms such as `IntegratedGradients, DeepLift,
GradientShap, NeuronConductance, LayerConductance, InternalInfluence and
NeuronIntegratedGradients`.

```
input = torch.rand(2, 3)
baseline = torch.zeros(2, 3)
```
Next we will use `IntegratedGradients` algorithms to assign attribution
scores to each input feature with respect to the second target output.
```
ig = IntegratedGradients(model)
attributions, delta = ig.attribute(input, baseline, target=1, return_convergence_delta=True)
print('IG Attributions: ', attributions, ' Convergence Delta: ', delta)
```
Output:
```
IG Attributions:  tensor([[0.0628, 0.1314, 0.0747],
                          [0.0930, 0.0120, 0.1639]])
Convergence Delta: tensor([0., 0.])
```
The algorithm outputs an attribution score for each input element and a
convergence delta that we would like to minimize. If we choose not to return
delta, we can simply not provide `return_convergence_delta` input argument.
The absolute value of the returned deltas can be interpreted as an approximation error
for each input sample.
It can also serve as a proxy of how much we can trust the attribution scores
assigned by an attribution algorithm.
If the approximation error is large, we can try larger number of integral
approximation steps by setting `n_steps` to a larger value. Not all algorithms
return approximation error. Those which do, though, compute it based on the
completeness property of the algorithms.

Positive attribution score means that the input in that particular position positively
contributed to the final prediction and negative means the opposite.
The magnitude of the attribution score signifies the strength of the contribution.
Zero attribution score means no contribution from that particular feature.

Similarly, we can apply `GradientShap`, `DeepLift` and other attribution algorithms to the model.

Gradient SHAP first chooses a random baseline from baselines' distribution, then
 adds gaussian noise with std=0.9 to each input example `n_samples` times.
Afterwards, it chooses a random point between each example-baseline pair and
computes the gradients with respect to target class (in this case target=0). Resulting
attribution is the mean of gradients * (inputs - baselines)
```
gs = GradientShap(model)

# We define a distribution of baselines and draw `n_samples` from that
# distribution in order to estimate the expectations of gradients across all baselines
baseline_dist = torch.randn(10, 3) * 0.001
attributions, delta = gs.attribute(input, stdevs=0.09, n_samples=4, baselines=baseline_dist,
                                   target=0, return_convergence_delta=True)
print('GradientShap Attributions: ', attributions, ' Convergence Delta: ', delta)
```
Output
```
GradientShap Attributions:  tensor([[ 0.0008,  0.0019,  0.0009],
                                    [ 0.1892, -0.0045,  0.2445]])
Convergence Delta: tensor([-0.2681, -0.2633, -0.2607, -0.2655, -0.2689, -0.2689,  1.4493, -0.2688])

```
Deltas are computed for each `n_samples * input.shape[0]` example. The user can,
for instance, average them:
```
deltas_per_example = torch.mean(delta.reshape(input.shape[0], -1), dim=1)
```
in order to get per example average delta


Below is an example of how we can apply `DeepLift` and `DeepLiftShap` on the
`ToyModel` described above. Current implementation of DeepLift supports only
`Rescale` rule.
For more details on alternative implementations, please read DeepLift's
original paper linked below.

```
dl = DeepLift(model)
attributions, delta = dl.attribute(input, baseline, target=0, return_convergence_delta=True)
print('DeepLift Attributions: ', attributions, ' Convergence Delta: ', delta)
```
Output
```
DeepLift Attributions:  tensor([[0.0628, 0.1314, 0.0747],
                                [0.0930, 0.0120, 0.1639]])
Convergence Delta: tensor([0., 0.])
```
DeepLift assigns similar attribution scores as Integrated Gradients to inputs,
however it has lower execution time. Another important thing to remember about
DeepLift is that it currently doesn't support all non-linear activation types.
For more details on limitations of the current implementation, please read
DeepLift's original paper linked below.

Similar to integrated gradients, DeepLift returns a convergence delta score
per input example. The approximation error is then the absolute
value of the convergence deltas and can serve as a proxy of trust for attribution
algorithms.

Now let's look into `DeepLiftShap`. Similar to `GradientShap`, `DeepLiftShap` uses
baseline distribution. In the example below, we use the same baseline distribution
as for `GradientShap`.

```
dl = DeepLiftShap(model)
attributions, delta = dl.attribute(input, baseline_dist, target=0)
print('DeepLiftSHAP Attributions: ', attributions, ' Convergence Delta: ', delta)
```
Output
```
DeepLift Attributions: tensor([0.0627, 0.1313, 0.0747],
                              [0.0929, 0.0120, 0.1637], grad_fn=<MeanBackward1>)
Convergence Delta:  tensor([-2.9802e-08,  0.0000e+00,  0.0000e+00,  0.0000e+00,  0.0000e+00,
         0.0000e+00,  0.0000e+00,  0.0000e+00,  0.0000e+00,  2.9802e-08,
         0.0000e+00,  0.0000e+00,  0.0000e+00,  0.0000e+00,  0.0000e+00,
         0.0000e+00,  0.0000e+00,  2.9802e-08,  0.0000e+00,  2.9802e-08])
```
`DeepLiftShap` uses `DeepLift` to compute attribution score for each
input-baseline pair and averages it for each input across all baselines.

It computes deltas for each input example-baseline pair, thus resulting to
`input.shape[0] * baseline.shape[0]` delta values.

Similar to GradientShap in order to compute example-based deltas we can average them per example:
```
deltas_per_example = torch.mean(delta.reshape(input.shape[0], -1), dim=1)
```
In order to smooth and improve the quality of the attributions we can run
`IntegratedGradients` and other attribution methods through a `NoiseTunnel`.
`NoiseTunnel` allows us to use SmoothGrad, SmoothGrad_Sq and VarGrad techniques
to smoothen the attributions by aggregating them for multiple noisy
samples that were generated by adding gaussian noise.

Here is an example how we can use `NoiseTunnel` with `IntegratedGradients`.

```
ig = IntegratedGradients(model)
nt = NoiseTunnel(ig)
attributions, delta = nt.attribute(input, nt_type='smoothgrad', stdevs=0.02, n_samples=4,
      baselines=baseline, target=0, return_convergence_delta=True)
print('IG + SmoothGrad Attributions: ', attributions, ' Convergence Delta: ', delta)
```
Output
```
IG + SmoothGrad Attributions:  tensor([[0.0631, 0.1335, 0.0723],
                                       [0.0911, 0.0142, 0.1636]])
Convergence Delta:  tensor([ 1.4901e-07, -8.9407e-08,  1.1921e-07,
        1.4901e-07,  1.1921e-07, -1.7881e-07, -5.9605e-08,  5.9605e-08])

```
The number of elements in the `delta` tensor is equal to: `n_samples * input.shape[0]`
In order to get a example-based delta, we can, for example, average them:
```
deltas_per_example = torch.mean(delta.reshape(input.shape[0], -1), dim=1)
```

Let's look into the internals of our network and understand which layers
and neurons are important for the predictions.

We will start with the neuron conductance. Neuron conductance helps us to identify
input features that are important for a particular neuron in a given
layer. It decomposes the computation of integrated gradients via the chain rule by
defining the importance of a neuron as path integral of the derivative of the output
with respect to the neuron times the derivatives of the neuron with respect to the
inputs of the model.

In this case, we choose to analyze the first neuron in the linear layer.

```
nc = NeuronConductance(model, model.lin1)
attributions = nc.attribute(input, neuron_index=1, target=0)
print('Neuron Attributions: ', attributions)
```
Output
```
Neuron Attributions:  tensor([[0.0000, 0.2854, 0.0000],
                              [0.0000, 0.1238, 0.0000]])
```

Layer conductance shows the importance of neurons for a layer and given input.
It is an extension of path integrated gradients for hidden layers and holds the
completeness property as well.

It doesn't attribute the contribution scores to the input features
but shows the importance of each neuron in selected layer.
```
lc = LayerConductance(model, model.lin1)
attributions, delta = lc.attribute(input, baselines=baseline, target=0, return_convergence_delta=True)
print('Layer Attributions: ', attributions, ' Convergence Delta: ', delta)
```
Outputs
```
Layer Attributions: tensor([[0.0891, 0.2758, 0.0476],
                            [0.0219, 0.1019, 0.3152]], grad_fn=<SumBackward1>)
Convergence Delta:  tensor([-0.0735, -0.0589])
```

Similar to other attribution algorithms that return convergence delta, LayerConductance
returns the deltas for each example. The approximation error is then the absolute
value of the convergence deltas and can serve as a proxy of trust for attribution
algorithms.

More details on the list of supported algorithms and how to apply
Captum on different types of models can be found in our tutorials.


## Captum Insights

Captum provides a web interface called Insights for easy visualization and
access to a number of our interpretability algorithms.

To analyze a sample model on CIFAR10 via Captum Insights run

```
python -m captum.insights.example
```

and navigate to the URL specified in the output.

![Captum Insights Screenshot](./website/static/img/captum_insights_screenshot.png)

To build Insights you will need [Node](https://nodejs.org/en/) >= 8.x
and [Yarn](https://yarnpkg.com/en/) >= 1.5.

To build and launch from a checkout in a conda environment run

```
conda install -c conda-forge yarn
BUILD_INSIGHTS=1 python setup.py develop
python captum/insights/example.py
```

## Contributing
See the [CONTRIBUTING](CONTRIBUTING.md) file for how to help out.


## References

* [Axiomatic Attribution for Deep Networks, Mukund Sundararajan et al. 2017](https://arxiv.org/abs/1703.01365)
* [Did the Model Understand the Question? Pramod K. Mudrakarta, et al. 2018](https://www.aclweb.org/anthology/P18-1176)
* [Investigating the influence of noise and distractors on the interpretation of neural networks, Pieter-Jan Kindermans et al. 2016](https://arxiv.org/abs/1611.07270)
* [SmoothGrad: removing noise by adding noise, Daniel Smilkov et al. 2017](https://arxiv.org/abs/1706.03825)
* [Local Explanation Methods for Deep Neural Networks Lack Sensitivity to Parameter Values, Julius Adebayo et al. 2018](https://arxiv.org/abs/1810.03307)
* [Sanity Checks for Saliency Maps, Julius Adebayo et al. 2018](https://arxiv.org/abs/1810.03292)
* [How Important is a neuron?, Kedar Dhamdhere et al. 2018](https://arxiv.org/abs/1805.12233)
* [Learning Important Features Through Propagating Activation Differences, Avanti Shrikumar et al. 2017](https://arxiv.org/pdf/1704.02685.pdf)
* [Computationally Efficient Measures of Internal Neuron Importance, Avanti Shrikumar et al. 2018](https://arxiv.org/pdf/1807.09946.pdf)
* [A Unified Approach to Interpreting Model Predictions, Scott M. Lundberg et al. 2017](http://papers.nips.cc/paper/7062-a-unified-approach-to-interpreting-model-predictions)
* [Influence-Directed Explanations for Deep Convolutional Networks, Klas Leino et al. 2018](https://arxiv.org/pdf/1802.03788.pdf)
* [Towards better understanding of gradient-based attribution methods for deep neural networks, Marco Ancona et al. 2018](https://openreview.net/pdf?id=Sy21R9JAW)

## License
Captum is BSD licensed, as found in the [LICENSE](LICENSE) file.
