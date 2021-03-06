# Oxford AIMS CDT Systems Verification (HT, 2020)

## Lab Session: DeepGame

This is the webpage for the lab session of the __[AIMS CDT](https://eng.ox.ac.uk/aims-cdt/) Systems Verification__ course. The lecture slides of this course can be found [here](https://github.com/TrustAI/DeepGame/blob/master/Oxford-AIMS-CDT-SystemsVerification-HT2020.pdf).

To better understand the tool __DeepGame__, please feel free to look into the accompanying published paper: 
[A game-based approximate verification of deep neural networks with provable guarantees](https://www.sciencedirect.com/science/article/pii/S0304397519304426).

In general, DeepGame _verifies_ the robustness of deep neural networks via a two-player turn-based _game_. It solves two problems &mdash; the _maximum safe radius_ problem in a _cooperative_ game and the _feature robustness_ problem in a _competitive_ game.

In this lab session, we primarily focus on the _maximum safe radius_ problem in a _cooperative_ game. Specifically, we look into three aspects: (1) search for adversarial examples, (2) generation of saliency maps, and (3) robustness guarantees of deep neural networks.

We start with the installation of the DeepGame tool.



### 0. Installation of DeepGame

Download the __DeepGame__ tool from https://github.com/minwu-cs/DeepGame. In order to run the tool, _Python_ and some other packages such as _keras_ and _numpy_ need to be installed. 

Below is a list of the developer's platform for reference in case there is package inconsistency.

###### Developer's Platform
```
python 3.5.5
keras 2.1.5
tensorflow 1.1.0
numpy 1.14.2
matplotlib 2.2.2
scipy 1.0.1
opencv-contrib-python 3.4.1.15
scikit-image 0.14.0
```

Use the following command line to run DeepGame. 

###### Run
```
python main.py mnist ub cooperative 67 L2 10 1
```
Here, `main.py` is the main Python file, in which you may find each parameter correspondingly.

```javascript
dataSetName = 'mnist'
bound = 'ub'
gameType = 'cooperative'
image_index = 67
eta = ('L2', 10)
tau = 1
```

Detailed explanation of each parameter is as follows.
- `dataSetName` refers to the name of the dataset. Currently DeepGame supports three image datasets `mnist`, `cifar10`, and `gtsrb`.
- `bound` denotes whether it is an upper bound `ub` or a lower bound `lb`.
- `gameType` indicates the type of the game. Specifically, a `cooperative` game is to compute the _maximum safe radius_ whereas a `competitive` game is to compute the _feature robustness_.
- `image_index` is the index of the image in the dataset.
- `eta` determines the distance metric and the distance budget. For the metric, `L2` is short for the _L<sup>2</sup> norm_, i.e., the Euclidean distance. Use `L1` for the _L<sup>1</sup> norm_ (Manhattan distance), or `L0` for the Hamming distance. In this case, `10` is a possible distance budget.
- `tau` denotes the value of _atomic manipulation_ imposed on each dimension of the input, i.e., each pixel/channel of an image.


Alternatively, you may run DeepGame via the following command line, which allows a sequential execution of the above command line.
```
./commands.sh
```
Within the `commands.sh` file, you may set the parameter values as needed.
```javascript
for i in {0..10}
do
    python main.py mnist ub cooperative $i L2 10 1
    python main.py mnist lb cooperative $i L2 0.01 1
done
exit 0
```

-------------------




### 1. Search for Adversarial Examples

An _adversarial example_ is an input which, though initially classified correctly by a neural network, is misclassified after a minor, perhaps imperceptible, perturbation. 

In DeepGame, this minor perturbation is set by the parameter `tau`, which imposes an _atomic manipulation_ on each pixel/channel of an input image. After pre-processing of the image datasets, all pixel values are normalised from [0,255] to [0,1], therefore we set the `tau` value from `(0,1]`.

To search for adversarial examples, we let the two players work in a _cooperative_ game and utilise the _Monte Carlo tree search_ algorithm. From the original image as the root, the game tree expands when the two players proceed in a turn-based way, where Player I chooses a feature of an input to perturb and then Player II determines the atomic manipulations within this chosen feature. 

The _termination condition_ for the game tree is that either an adversarial example is found or a distance budget based on a certain metric `eta` is reached. Note that the _distance budget_ should be a reasonable value because if perturbations imposed on the input are too much to the extent that even humans are not able to distinguish, then it is no longer sensible to require a neural network to classify correctly.  

When the execution of DeepGame proceeds, improved adversarial examples in the sense of with fewer and fewer modifications are generated.

#### Questions: 

> 1. Produce some adversarial examples on the MNIST dataset via utilising the _Monte Carlo tree search_ algorithm.
> _Requirements: (1) try image index 67 of the MNIST dataset; (2) based on the L<sup>1</sup> norm and set the distance budget as 10; (3) let the atomic manipulation value be 1._


Below are some adversarial examples of the MNIST, CIFAR-10, and GTSRB datasets when the metric is the L<sup>2</sup> norm.

![alt text](figures/Adversary.png)




<!--- 
Whereas DeepGame is primarily a _robustness verification_ tool, with slight modification in the code, it can be adapted to perform _adversarial attacks_ on the image datasets. For instance, every upper bound produced from the Monte Carlo tree search algorithm contributes to an adversarial example.

Below, we perform efficient adversarial attacks through changing the Admissible A* algorithm to the _Inadmissible A*_ variant, by increasing the `heuristic` value added to the actual `cost` in the `CooperativeAStar.py` file.

```javascript
cost = self.cal_distance(manipulated_images[idx], self.IMAGE)
[p_max, p_2dn_max] = heapq.nlargest(2, probabilities[idx])
heuristic = (p_max - p_2dn_max) * 2 * self.TAU  # heuristic value
estimation = cost + heuristic
```
#### Questions: 
> 5. Produce some adversarial examples on the MNIST, the CIFAR10, and the GTSRB datasets, via utilising the _Inadmissible A*_ algorithm.
> _Requirements: (1) try images from the three datasets with index from 0 to 99; (2) based on the Hamming distance, the L<sup>1</sup> norm, or the L<sup>2</sup> norm._

> 6. Explain why increasing the heuristic value would make the A* algorithm no longer admissible.
-->

-------------------





### 2. Generation of Saliency Maps

To facilitate the explainability and the interpretability of the deep neural networks, DeepGame can generate the _saliency map_ of an input point, to better demonstrate how a network model actually 'sees' or 'understands' an image.

To generate the saliency map, DeepGame utilises two _feature extraction_ methods to partition the input into a disjoint set of dimensions with varying saliency levels. One method is called the _grey-box_ feature extraction, which exploits the output logits/probabilities of a classifier to discover the more sensitive input dimensions, whereas the other approach is _black-box_ as it does not reply on networks but instead directly applies the scale-invariant feature transform (SIFT) on the input locally.

In DeepGame, these feature extraction methods are defined in the `FeatureExtraction.py` file. You may set a specific `grey-box` or `black-box` pattern via the following code line in the `GameMoves.py` file.

```javascript
feature_extraction = FeatureExtraction(pattern='grey-box')
```
In order to output the heatmap of the saliency map, you need to uncomment these two lines in `GameMoves.py`.

```javascript
# path = "%s_pic/%s_Saliency_(%s).png" % (self.data_set, self.image_index, feature_extraction.PATTERN)
# feature_extraction.plot_saliency_map(self.image, partitions=partitions, path=path)
```        


#### Questions: 

> 2. Generate a saliency map of an image with the _grey-box_ feature extraction method.
> _Requirements: (1) try the same MNIST image with index 67; (2) compare the saliency map with the original image._

> 3. Change the feature extraction method from grey-box to _black-box_, and generate another saliency map of the image.
> _Requirements: (1) use the same MNIST image with index 67 and keep the other parameters unchanged; (2) compare these two saliency maps with the original image._

Below exhibits two saliency maps of a CIFAR-10 image and a GTSRB image from the grey-box extraction method.

![alt text](figures/Feature.png)

-------------------





### 3. Robustness Guarantees of Deep Neural Networks

Apart from searching for adversarial examples and generating saliency maps, our tool DeepGame is more importantly a verification tool that can _verify deep neural networks with provable guarantees_. 

Regarding this, we compute the _maximum safe radius of a neural network with respect to an input_, which is a distance such that, with imposed perturbations below the distance, all the input points are safe, whereas if above the distance, there definitely exists an adversarial example. 

While the number of possible input points is infinite, the maximum safe radius can be approximated by descretising the input space under the assumption of _Lipschitz continuity_. Intuitively, it is sufficient to consider a finite number of uniformly sampled inputs on the grid when the distance between the grid points is small. Here, the distance between grid points is reflected in the parameter `tau`, which sets the atomic manipulation.

As computing the maximum safe radius directly is NP-hard, we compute the _lower and upper bounds_ of it and show the convergence trend. In other words, every _lower bound_ guarantees that all the possible perturbations below it are safe, while every _upper bound_ indicates the existence of an adversarial example. That said, every adversarial example generated from the _Monte Carlo tree search_ in Question 1 contributes to an upper bound. As for the lower bound, DeepGame utilises the _Admissible A*_ algorithm in a cooperative game and the _Alpha-Beta Pruning_ algorithm in a competitive game.


#### Questions: 
> 4. Plot a figure to illustrate the convergence of the lower and upper bounds of the maximum safe radius. 
> _Requirements: (1) try the same MNIST image with index 67; (2) based on the Euclidean distance; (3) for the upper bound, set the distance budget as 10; (4) for the lower bound, set the distance budget as 0.01; (5) set the atomic manipulation as 1; (6) use grey-box feature extraction method._

> 5. Exhibit some safe perturbations imposed on the original image corresponding to the lower bounds, and also some adversarial examples generated as a by-product when computing the upper bounds.

> 6. Change the value of _atomic manipulation_ in the range of `(0,1]`, for example, `0.5`, `0.05`, or `0.005`, and observe its influence on the lower bound.

> 7. Evaluate the robustness of a neural network trained on another dataset. Plot the convergence of the bounds and display the safe and unsafe adversarial perturbations.
> _Requirements: (1) try an image from the CIFAR-10 dataset with index from 0 to 99._


Below suggests a possible solution to the above Questions 4 and 5.

![alt text](figures/Cooperative_MNIST.png)

-------------------





### Citation of DeepGame:

Below is the citation of the accompanying paper:

[A game-based approximate verification of deep neural networks with provable guarantees](https://www.sciencedirect.com/science/article/pii/S0304397519304426).

```
@article{wu2020game,
  title   = "A Game-Based Approximate Verification of Deep Neural Networks with Provable Guarantees",
  author  = "Wu, Min and Wicker, Matthew and Ruan, Wenjie and Huang, Xiaowei and Kwiatkowska, Marta",
  journal = "Theoretical Computer Science",
  volume  = "807",
  pages   = "298 - 329",
  year    = "2020",
  note    = "In memory of Maurice Nivat, a founding father of Theoretical Computer Science - Part II",
  issn    = "0304-3975",
  doi     = "https://doi.org/10.1016/j.tcs.2019.05.046",
  url     = "http://www.sciencedirect.com/science/article/pii/S0304397519304426"
}
```





### Remark

This webpage is for the lab session of the AIMS CDT Systems Verification course. Should you have any questions, please feel free to contact the teaching assistant __Min Wu__ via min.wu@cs.ox.ac.uk.

Best wishes,

Min
