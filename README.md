# Haiku

**Haiku is Sonnet for JAX.**

Haiku is a simple neural network library for
[JAX](https://github.com/google/jax) developed by some of the
authors of [Sonnet](https://github.com/deepmind/sonnet), a neural network
library for TensorFlow.

NOTE: Haiku is currently **beta**. A number of researchers have tested Haiku
for several months and have reproduced a number of experiments at scale. Please
feel free to use Haiku, but be sure to test any assumptions and to
[let us know](https://github.com/deepmind/haiku/issues) if things don't look
right!

## Overview

[JAX](https://github.com/google/jax) is a numerical computating library that
combines NumPy, automatic differentiation, and first-class GPU/TPU support.

Haiku is a simple neural network library for JAX that enables users to use
familiar **object-oriented programming models** while allowing full access to
JAX's pure function transformations.

Haiku provides two core tools: a module abstraction, `hk.Module`, and a simple
function transformation, `hk.transform`.

`hk.Module` behaves like `snt.Module`: `Module`s are Python objects that hold
references to their own parameters, other modules, and methods that apply
functions on user inputs.

`hk.transform` turns functions that use these object-oriented, functionally
"impure" modules into pure functions that can be used with `jax.jit`,
`jax.grad`, `jax.pmap`, etc.

## Why Haiku?

There are a number of neural network libraries for JAX. Why should you choose
Haiku?

### Haiku does not reinvent the wheel.

- Haiku builds on the programming model and APIs of Sonnet, a neural network
  library with near universal adoption at DeepMind. It preserves Sonnet's
  `Module`-based programming model for state management while retaining access
  to JAX's function transformations.
- Haiku APIs and abstractions are as close as reasonable to Sonnet. Many users
  have found Sonnet to be a productive programming model in TensorFlow; Haiku
  enables the same experience in JAX.

### Transitioning to Haiku is easy.

- By design, transitioning from TensorFlow and Sonnet to JAX and Haiku is easy.
- Outside of new features (e.g. `hk.transform`), Haiku aims to match the API of
  Sonnet 2. Modules, methods, argument names, and defaults, and initialization
  schemes should match.

### Haiku has been tested by researchers at DeepMind at scale.

- DeepMind has reproduced a number of experiments in Haiku and JAX with relative
  ease, thanks to Haiku's API similarity with Sonnet.
- These include large-scale results in image and language processing, generative
  models, and reinforcement learning.

### Haiku is a library, not a framework.

- Haiku (and Sonnet) are designed to make specific things simpler: managing
  model parameters and other model state.
- Haiku can be expected to compose other libraries and work well with the rest
  of JAX.
- Haiku otherwise is designed to get out of your way - it does not define custom
  optimizers, checkpointing formats, or replication APIs.

### Haiku makes other aspects of JAX simpler.

- Haiku offers a trivial model for working with random numbers. Within a
  transformed function, `hk.next_rng_key()` returns a unique rng key.
- These unique keys are deterministically derived from an initial random key
  passed into the top-level transformed function, and are thus safe to use with
  JAX program transformations.

## Quickstart

Let's take a look at an example neural network and loss function. This looks
basically the same as in TensorFlow with Sonnet:

```python
import haiku as hk
import jax.numpy as jnp

def loss_fn(images, labels):
  model = hk.Sequential([
      hk.Linear(1000), jax.nn.relu,
      hk.Linear(100), jax.nn.relu,
      hk.Linear(10),
  ])
  logits = model(images)
  labels = hk.one_hot(labels, 10)
  return jnp.mean(softmax_cross_entropy(logits, labels))

loss_obj = hk.transform(loss_fn)
```

`hk.transform` allows us to look at this function in two ways. First, it allows
us to run the function and collect initial values for parameters:

```python
rng = jax.random.PRNGKey(42)
images, labels = next(input_dataset)  # Example input.
params = loss_obj.init(rng, images, labels)
```

Second, it allows us to run the function and compute the output, but explicitly
passing in parameter values:

```python
loss = loss_obj.apply(params, images, labels)
```

This is useful since we can now take gradients of the loss with respect to the
parameters:

```python
grads = jax.grad(loss_obj.apply)(params, images, labels)
```

Which allows us to write a simple SGD training loop:

```python
def sgd(param, update):
  return param - update * 0.01

for _ in range(num_training_steps):
  images, labels = next(input_dataset)
  grads = jax.grad(loss_obj.apply)(params, images, labels)
  params = jax.tree_multimap(sgd, params, grads)
```

For more, see our examples directory. The
[MNIST example](https://github.com/deepmind/haiku/tree/master/examples/mnist.py)
is a good place to start.

## User manual

### Writing your own modules

In Haiku, all modules are a subclass of `hk.Module`. You can implement any
method you like (nothing is special-cased), but typically modules implement
`__init__` and `__call__`.

Let's work through implementing a linear layer:

```python
class MyLinear(hk.Module):

  def __init__(self, output_size, name=None):
    super(MyLinear, self).__init__(name=name)
    self.output_size = output_size

  def __call__(self, x):
    j, k = x.shape[-1], self.output_size
    w_init = hk.initializers.TruncatedNormal(1. / np.sqrt(j))
    w = hk.get_parameter("w", shape=[j, k], dtype=x.dtype, init=w_init)
    b = hk.get_parameter("b", shape=[k], dtype=x.dtype, init=jnp.zeros)
    return jnp.dot(x, w) + b
```

In Haiku all modules have a name. Modules can also have named parameters that
are accessed using `hk.get_parameter(param_name, ...)`. We use this API (rather
than just using object properties) so that we can convert your code into a pure
function using `hk.transform`.

When using modules you need to define functions and transform them using
`hk.transform`. This function
wraps your function into an object that provides `init` and `apply` methods.
These run your original function under writer/reader monads allowing us to
collect and inject parameters, state (e.g. batch stats) and rng keys:

```python
def forward_fn(x):
  model = MyLinear(10)
  return model(x)

# Turn `forward_fn` into an object with `init` and `apply` methods.
forward = hk.transform(forward_fn)

x = jnp.ones([1, 1])

# When we run `forward.init`, Haiku will run `forward(x)` and collect initial
# parameter values. By default Haiku requires you pass a RNG key to `init`,
# since parameters are typically initialized randomly:
key = hk.PRNGSequence(42)
params = forward.init(next(key), x)

# When we run `forward.apply`, Haiku will run `forward(x)` and inject parameter
# values from what you pass in. We do not require an RNG key by default since
# models are deterministic. You can (of course!) change this using
# `hk.transform(f, apply_rng=True)` if you prefer:
y = forward.apply(params, x)
```

### Working with non-trainable state

TODO(tomhennigan): Write me!

### Distributed training with `jax.pmap`

TODO(tomhennigan): Write me!