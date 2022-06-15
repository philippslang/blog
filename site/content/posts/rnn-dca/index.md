---
title: Decline curve modeling using recurrent neural networks
description: Use of RNN to model production decline curves that account for fracking events.
date: 2017-07-26
tldr: Feeding a binary signal to indicate fracking on production decline curves in RNNs.
tags: ["ML models", "energy"]
---

- Conventional decline curve models cannot account for changing conditions
- Recurrent neural networks (RNN) process temporal data
- Temporal data may be used to represent changes in conditions during a well's lifetime
- This provides means for a new decline curve modeling framework
- This new framework is able to account for refracking, well interactins and more

Decline curve modeling is arguably one of the least challenging problems to throw at neural networks these days. So now that I've sufficiently lowered the bar, here goes a short write-up of a proof of concept implementation I recently demoed at a seminar. Proof of concept implementations are great since we don't have to put up with users and/or real data.

Shale wells are prime candidates for decline curve modeling. Refracking of a shale well, however, breaks the underlying assumption of decline curve modeling: first order controls which govern the production history may not change during the forecast period. Let's assume we have a set of production profiles like this, which we know for a fact to be representative:

{{< figure src="./images/profile_example.svg" caption="An artificial production history with stage information (binary array here). We'll use profiles like these to train our network." >}}

We see two stages, zero and one, where the stage transition indicates the point in time of refracking. 
This post is about predicting the production profile given some initial production data and the planned 
time of refrecking, which should allow for comparisons as to the economically most sensible point for refracturing, for example. We'll be moving through this rather swiftly, without much regard for Python or Keras fundamentals (yes, naturally we'll be using Keras here). Let's get to it then - we'll start with the import of some of the usual suspects:

<script src="https://gist.github.com/plang85/5b63fbd837b608ac3e50583ac2de1b63.js"></script>

where we also define a file name under which we'll save the trained model for fast predicitons, like when you're making figures for a blog post. Next we define a function to produce a production profile like the one above:

<script src="https://gist.github.com/plang85/845141802581bbad8117ade85b490883.js"></script>

Let's not dwell over the details of this function. Suffice it to say that it provides means to generate above 
production profiles, whith two stages of different exponents, and some Gaussian noise. For convenience 
we'll use this function to generate training data for us. The model we are going to use is defined by

<script src="https://gist.github.com/plang85/d0dab63233acc7775119283230c175c6.js"></script>

where `kel.TimeDistributed(kel.Dense(num_targets))` forms a dense operation on the outputs for a single time
step from all units. In other words, it creates `num_timesteps` fully connected layers with `num_target = 1` neurons, each with `num_units` inputs/weights. Let's be explicit about the layout of our little network and some convenience indices

<script src="https://gist.github.com/plang85/8ab00751dbc7a66cc3e477326392cd69.js"></script>

where the features we provide are the production history and the stage array (zeros and ones for stage zero and one, respectively) of the production profiles. The targets which we train against is the entire production history of the profile (without the first value, but more on this later). We go on and define the training data setup and array layouts

<script src="https://gist.github.com/plang85/23a8acee4ae83c87b6b831e3942b270e.js"></script>

Filling the arrays

<script src="https://gist.github.com/plang85/2fbeff720b23e6141887fbbc508baf28.js"></script>

The generated production profiles that form the basis for our feature and target arrays for our training data set look like this

{{< figure src="./images/training_data.svg" caption="Generated training data. Not shown are the corresponding stage arrays." >}}

For each of these profiles, we generate sequences for all time steps except the last one. Each progressing sequence has one more defined productino value. The corresponding target production is always the full profile, except for the first value which is `na`-encoded, since this value is part of all feature sequences.

{{< figure src="./images/seq010_data.svg" caption="NA encoding in a sequence in our feature array" >}}

If the above sequence is the 10th of a given production profile, the 30th would look like this

{{< figure src="./images/seq030_data.svg" caption="NA encoding in a sample in our feature array, for a later sequence of the same profile." >}}

and so on until the next to last production value. Each sample in a recurrent network training deck is in fact a sequence, with `num_timesteps` values per feature.

Next it is good practice to normalize the feature arrays - this keeps the weights of the input layers closer to unity and thus aids convergence and somewhat mitigates FP rounding errors. It is required, however, to normalize the target arrays if the activation of the output layer maps to a bounded interval, which is the case for the `sigmoid` function we use. Our scaling is thus implicitly linked with the chosen activation, something to keep in mind and something that should be expressed more explictily in a proper application. The scalers available in `sklearn.preprocessing` require a two-dimensional view of the underlying values, with samples as rows and each feature/target as columns, so this view is exactly what we provide by flattening the time step dimension of our arrays:

<script src="https://gist.github.com/plang85/ad9f1bd7ba7dda5ddf6fb2874c85d35b.js"></script>

Let's train the model with some esoteric values for the number of epochs and batch size - just remember the a single profile has `num_timesteps - 1` sequences, so I would keep the `batch_size` here below that to get at least one weight update per profile. We add hack to load a trained model for future quick predictions and off we go.

<script src="https://gist.github.com/plang85/c7d8f7969920d53f7092e07d2bd6f6ac.js"></script>

Running the script you should see something along these lines (I'm on nothing fancy here, some ThinkPad on some i5 running Fedora, no GPU or what have you, also TF keeps telling me to recompile for the little extra instruction sets I have):

```
1960/1960 [==============================] - 3s - loss: 0.0133 - val_loss: 0.0101
Epoch 2/10
1960/1960 [==============================] - 2s - loss: 0.0094 - val_loss: 0.0087
Epoch 3/10
1960/1960 [==============================] - 2s - loss: 0.0084 - val_loss: 0.0077
Epoch 4/10
1960/1960 [==============================] - 2s - loss: 0.0074 - val_loss: 0.0065
Epoch 5/10
1960/1960 [==============================] - 2s - loss: 0.0058 - val_loss: 0.0045
Epoch 6/10
1960/1960 [==============================] - 2s - loss: 0.0035 - val_loss: 0.0023
Epoch 7/10
1960/1960 [==============================] - 2s - loss: 0.0017 - val_loss: 0.0012
Epoch 8/10
1960/1960 [==============================] - 2s - loss: 9.4538e-04 - val_loss: 7.1189e-04
Epoch 9/10
1960/1960 [==============================] - 2s - loss: 6.4418e-04 - val_loss: 5.3728e-04
Epoch 10/10
1960/1960 [==============================] - 2s - loss: 5.0730e-04 - val_loss: 4.3987e-04
```

The continuous residual reduction is encouraging, as is the three orders of magnitude decrease in loss. This indicates a well behaved model, and indeed the function generated above should be something to easily regress against. We can now use our model for prediction

<script src="https://gist.github.com/plang85/8722b3860b261a64816d40188193822b.js"></script>

Upon inspection of the de-normalized (remember the 2D view thing) target array, the prediction sequence should look like this:

```
[-39.26998901  41.88949966  38.46153259  33.63489151  28.81029129
  25.71617889  22.57770538  19.67162323  16.9606514   14.97911167
  13.07455444  11.59985352  10.24969101   9.04060364   7.96830654
   7.02176762   6.18764973   5.45265293  16.6064167   15.20786476
  13.66429615  12.14694881  10.71009541   9.40864754   8.25721645
   7.25116396   6.3779912    5.62249804   4.96940517   4.40454197
   3.91531825   3.49083686   3.12173867   2.80009866   2.51921129
   2.27336454   2.05775905   1.86830044   1.70152593   1.55444443
   1.4245491    1.30965495   1.20788932   1.11762822   1.03748643
   0.96625328   0.90285933   0.8464036    0.79607713   0.75117487]
```

Where the first entry is near out `na` value, which reflects the way we set up our training sequences. This can now be plotted to illustrate the prediction based on the eight historic data points.

{{< figure src="./images/prediction_example.svg" caption="The predicion of our trained network given an initial production history." >}}

We can visually inspect what the above value for loss on the validation set looks like

{{< figure src="./images/prediction_reference_example.svg" caption="Comparing the prediction to the reference history." >}}

As a concluding note it is worth pointing out that stage changes don't need to result in an imediate production response. LSTMs were built to learn long-term correlations and should be well capable to model effects such as water injection in nearby wells, or the effect of a newly drilled well, provided appropriate training sets. These training data sets may be generated by a conventional simulator. The effect of stage changes in these cases won't affect the production profile for possibly long periods of time.
