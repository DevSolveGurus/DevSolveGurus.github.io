---
title : Addressing Discrepancies Between Keras and scikit-learn in Basic Feedforward Neural Network Implementation 
date : 2023-09-13 21:50:35 +0000
published: true  
---
# Addressing Discrepancies Between Keras and scikit-learn in Basic Feedforward Neural Network Implementation

I've created a fully-connected neural network using both scikit-learn version 0.20.0 and Keras version 2.2.4, utilizing the TensorFlow backend version 1.12.0. Each network consists of a single hidden layer with 10 units. In both instances, I split the training and test data using scikit-learn's train_test_split function, setting the random_state to 0 for consistency. Subsequently, I applied scikit-learn's StandardScaler to scale both datasets. Remarkably, the code for these initial steps is virtually identical.

For the scikit-learn implementation, I defined the neural network using MLPRegressor. The output of this function call is as follows:

```python
MLPRegressor(activation='logistic', alpha=1.0, batch_size='auto', beta_1=0.9,
   beta_2=0.999, early_stopping=False, epsilon=1e-08,
   hidden_layer_sizes=(10,), learning_rate='constant',
   learning_rate_init=0.001, max_iter=200, momentum=0.9,
   n_iter_no_change=10, nesterovs_momentum=True, power_t=0.5,
   random_state=None, shuffle=True, solver='sgd', tol=0.0001,
   validation_fraction=0.2, verbose=False, warm_start=False)

```
Many of these parameters go unused, but a subset of key parameters includes 200 iterations, no early stopping, a fixed learning rate, SGD as the solver, with nesterovs_momentum set to True, and a momentum value of 0.9.

In Keras, this configuration is referred to as "Keras 1."

```python
mlp = Sequential() # create a sequential neural network using Keras
mlp.add(Dense(units=10,activation='sigmoid',input_dim=X.shape[1],
          kernel_regularizer=skl_norm))
mlp.add(Dense(units=1,activation='linear'))
opt = optimizers.SGD(lr=0.001,momentum=0.9,decay=0.0,nesterov=True)
mlp.compile(optimizer=opt,loss='mean_squared_error')
mlp.fit(X_train,y_train,batch_size=200,epochs=200,verbose=0)
```
As I comprehend it, in Keras, this network should be identical to the scikit-learn version, with a single potential deviation: scikit-learn applies regularization to all weights connecting layers, whereas the Keras network only applies regularization to the weights from the input layer to the hidden layer. To include regularization for the weights from the hidden layer to the output layer, you can do so as follows, denoting this configuration as "Keras 2."

```python
mlp = Sequential() # create a sequential neural network using Keras
mlp.add(Dense(units=10,activation='sigmoid',input_dim=X.shape[1],
          kernel_regularizer=skl_norm))
mlp.add(Dense(units=1,activation='linear',kernel_regularizer=skl_norm))
opt = optimizers.SGD(lr=0.001,momentum=0.9,decay=0.0,nesterov=True)
mlp.compile(optimizer=opt,loss='mean_squared_error')
mlp.fit(X_train,y_train,batch_size=200,epochs=200,verbose=0)
```

To ensure that the regularization in Keras aligns with that of [scikit-learn](https://scikit-learn.org/stable/modules/sgd.html#mathematical-formulation), I've created a custom regularization function in Keras:

```python
def skl_norm(weight_matrix):
    alpha = 1.0 # to match parameter I used in sci-kit learn
    return alpha * 0.5 * K.sum(K.square(weight_matrix))
```

To ensure that regularization in Keras aligns with scikit-learn, I've implemented a custom regularization function in Keras. The alpha parameter should match what's used in scikit-learn. The code following these definitions differs only in the method names used by each API.

However, my results indicate that regularization is not consistent between the two APIs. It's possible that my Keras implementation is not behaving as expected. Here's a comparison of the neural network outputs:

Comparison between scikit-learn and Keras networks:

- The top row corresponds to alpha = 0, and the bottom row to alpha = 1.0.
- The left column represents scikit-learn, the middle column is Keras 1, and the right column is Keras 2.

What immediately stands out is that when regularization is "turned off" (alpha=0), the fits are very similar. However, when regularization is "turned on" (alpha=1), scikit-learn outperforms Keras, especially Keras 2 when the outputs of the hidden layer are regularized.

During different runs, the R^2 values vary slightly but not significantly enough to explain the differences in the bottom row. So, what distinguishes these two network implementations?

An update:

I've discovered that if I use an "unbounded" activation function in Keras, the training fails entirely, returning 'nan' for all predictions, whereas it works fine in scikit-learn. By "unbounded," I mean an activation function that allows output values of infinity, such as linear/identity, softplus, or relu.

When I enable the TensorBoard callback, I encounter an error that concludes with:

```bash
InvalidArgumentError (see above for traceback): Nan in summary histogram for: dense_2/bias_0 [[node dense_2/bias_0 (defined at /Users/.../python2.7/site-packages/keras/callbacks.py:796) = HistogramSummary[T=DT_FLOAT, _device="/job:localhost/replica:0/task:0/device:CPU:0"](dense_2/bias_0/tag, dense_2/bias/read)]]
```

Based on this error, it appears that the bias units for the second layer are becoming exceptionally large. However, I'm unsure why this occurs in Keras/TF but not in scikit-learn.

Since the softplus activation function doesn't have the property that f(x)=0 when x=0, I don't believe the issue is due to inputs clustering near zero. Furthermore, a tanh activation function works well. So, I don't think there's an issue with inputs approaching negative infinity. Both sigmoid/logistic and softplus have the property f(x)=0 when x->-infinity, and sigmoid/logistic works well while softplus fails. Therefore, I don't believe the problem stems from inputs approaching negative infinity.

## The Answer

Resolving Differences Between Keras and scikit-learn for a Fully-Connected Neural Network

When building a fully-connected neural network in both scikit-learn and Keras with TensorFlow, you may encounter differences in results. Below are some steps and considerations to resolve these differences:

#### 1. Regularization Implementation:

In scikit-learn, MLPRegressor applies regularization to all layers by default. In Keras, you need to specify regularization for each layer explicitly. Use Keras's kernel_regularizer parameter for this purpose:

```python
from keras import regularizers

# In Keras 2 (Apply regularization to both layers)
mlp = Sequential()
mlp.add(Dense(units=10, activation='sigmoid', input_dim=X.shape[1], kernel_regularizer=regularizers.l2(alpha)))
mlp.add(Dense(units=1, activation='linear', kernel_regularizer=regularizers.l2(alpha)))
```

#### 2. Activation Function and Exploding Gradients:

Avoid **"unbounded"** activation functions like **linear** or **softplus** in Keras, as they can lead to NaN values. Instead, use bounded activation functions such as sigmoid or tanh:


```python
# Use bounded activation functions in Keras
mlp.add(Dense(units=10, activation='sigmoid', input_dim=X.shape[1], kernel_regularizer=regularizers.l2(alpha)))
mlp.add(Dense(units=1, activation='linear', kernel_regularizer=regularizers.l2(alpha)))

```
#### 3. Weight Initialization:

Ensure consistent weight initialization methods. Use Glorot (Xavier) initialization in Keras as an example:
```python
from keras.initializers import glorot_uniform

# Set weight initialization method
mlp.add(Dense(units=10, activation='sigmoid', input_dim=X.shape[1], kernel_initializer=glorot_uniform(), kernel_regularizer=regularizers.l2(alpha)))
mlp.add(Dense(units=1, activation='linear', kernel_initializer=glorot_uniform(), kernel_regularizer=regularizers.l2(alpha)))
```
#### 4. Optimizer Settings:

Keep optimizer settings similar in both libraries. For instance, set the learning rate and momentum:

```python
from keras import optimizers

opt = optimizers.SGD(lr=0.001, momentum=0.9, decay=0.0, nesterov=True)
mlp.compile(optimizer=opt, loss='mean_squared_error')

```
#### 5. Numerical Precision:

Set the numerical precision explicitly to **float32** in Keras to match scikit-learn if necessary:

```python
import keras.backend as K
K.set_floatx('float32')

```

#### 6. Learning Rate Schedules:

Ensure both libraries use similar learning rate schedules if applicable. Handle changes in learning rate consistently.

#### 7. Random Initialization:

Set a random seed for both libraries to start with the same initial weights.

#### 8. Monitoring Training:

Use tools like TensorBoard for Keras to monitor training. Visualize training loss, gradients, and weights to identify discrepancies.

By aligning the configurations and settings between scikit-learn and Keras/TensorFlow, you should achieve similar results.

