---
id: "01M6EJDSNBMM4DKNDGF4PX1EYM"
title_en: "Backpropagation: Gradients via the Chain Rule and MNIST in NumPy"
summary_en: "Part 2 of AI Note. We follow backpropagation through the chain rule and computational graphs, starting from numeric examples, derive the gradients of a two-layer network in matrix form, and implement it from scratch in NumPy to reach 97.7% on MNIST. We also measure why minibatches are fast."
---
:::note
**The 3 key points of this post**
- Gradient descent needs the slope (gradient) of every weight. Nudging the weights one at a time and measuring would take 200,000 forward passes for 100,000 weights, but backpropagation gets them all with one forward pass plus one pass of similar cost.
- Backpropagation applies the chain rule of differentiation backward, starting from the loss. Each operation only has to "multiply the gradient it receives by its own derivative and pass it on."
- Writing a 784-128-10 network by hand in NumPy and training it on handwritten digits (MNIST) reaches 97.7% test accuracy in 1.6 seconds on a Mac M1 Max CPU.
:::

This is the second post in the AI Note series. In [Part 1](/en/post/neural-network-basics/), we said a neural network is a function built by stacking neurons in layers, and training is reducing the loss with gradient descent. But to do gradient descent, we need to know, **for every weight**, "how much does the loss change if I change this weight a little," that is, the gradient $\partial L / \partial w$. This post walks step by step through backpropagation, the efficient way to compute it, starting from small numeric examples, then implements it and trains on real data.

## 1. Why We Need Backpropagation

The simplest approach is to nudge each weight by a tiny amount (numerical differentiation).

$$\frac{\partial L}{\partial w} \approx \frac{L(w + \epsilon) - L(w - \epsilon)}{2\epsilon}$$

This is accurate but expensive. The network we build in this post has 101,770 weights. Each weight needs the loss computed twice, so **taking a single step requires about 200,000 forward passes**. Backpropagation gets the same result with 1 forward pass and 1 backward pass of similar cost. That is a 100,000x difference. Without backpropagation, deep learning as we know it would not be possible.

## 2. Starting Point: The Chain Rule

Backpropagation is not new math. It is a systematic way to use the **chain rule** taught in high school. When functions are nested, the derivative is the product of the derivatives of each step.

$$y = g(u),\; u = h(x) \quad\Rightarrow\quad \frac{dy}{dx} = \frac{dy}{du}\cdot\frac{du}{dx}$$

For example, let's differentiate $y = (3x + 1)^2$ at $x = 1$. Let $u = 3x + 1$, so $y = u^2$.

| Step | Value ($x = 1$) | Local derivative |
|---|---|---|
| $u = 3x + 1$ | $u = 4$ | $du/dx = 3$ |
| $y = u^2$ | $y = 16$ | $dy/du = 2u = 8$ |
| Chain rule | | $dy/dx = 8 \times 3 = \mathbf{24}$ |

Expanding to check: $y = 9x^2 + 6x + 1$, $y' = 18x + 6 = 24$, the same. There are two important observations here. First, each step only needs to know **the derivative of its own operation (the local derivative)**. Second, computing $dy/du = 2u$ **requires $u = 4$ from the forward pass.** These two points are all there is to backpropagation.

## 3. Viewing It as a Computational Graph

Breaking an expression into individual operations connected by arrows gives a computational graph. Let's compute $f = (x + y)\cdot z$ at $x = 2,\ y = -1,\ z = 3$.

![Computational graph example. The forward pass computes q = x + y = 1 and f = q × z = 3, and the backward pass starts from f to get ∂f/∂q = 3, ∂f/∂z = 1, ∂f/∂x = 3, ∂f/∂y = 3|full](/img/01M6EJDSNBMM4DKNDGF4PX1EYM/3ad7166c4f07.webp)

1. **Forward pass** (left → right): $q = x + y = 1$, $f = q \cdot z = 3$.
2. **Backward pass** (right → left): the starting point is $\partial f / \partial f = 1$.
   - Multiplication $f = q \cdot z$: $\partial f / \partial q = z = 3$, $\partial f / \partial z = q = 1$. Multiplication passes on **the received gradient times the other input's value**.
   - Addition $q = x + y$: since $\partial q / \partial x = \partial q / \partial y = 1$, it passes the received gradient 3 **unchanged** to both inputs. So $\partial f / \partial x = \partial f / \partial y = 3$.

Check: changing $x$ from 2 to 2.01 gives $f = 1.01 \times 3 = 3.03$, an increase of 0.03. A change of 0.01 yields 0.03, so the slope of 3 is correct.

Remember the rules from this example and any complicated expression can be solved the same way.

| Operation | What it does in the backward pass |
|---|---|
| Addition | Passes the received gradient unchanged to every input |
| Multiplication | Multiplies the received gradient by the other input's value |
| ReLU $\max(0, z)$ | Passes it through if $z > 0$ in the forward pass, otherwise 0 |
| Same value used in several places | Adds up the gradients returning from each path |

## 4. One Training Step with a Single Neuron

Now let's train a real neuron for one step. Input $x = 2$, weight $w = 0.5$, bias $b = 0.5$, ReLU activation, target $t = 3$, and squared-error loss $L = (a - t)^2$.

**Forward pass**

| Step | Computation | Value |
|---|---|---|
| Weighted sum | $z = wx + b = 0.5 \times 2 + 0.5$ | 1.5 |
| Activation | $a = \max(0, z)$ | 1.5 |
| Loss | $L = (a - t)^2 = (1.5 - 3)^2$ | 2.25 |

**Backward pass** (backward from the loss)

| Gradient | Computation | Value |
|---|---|---|
| $\partial L / \partial a$ | $2(a - t)$ | −3 |
| $\partial L / \partial z$ | $\partial L/\partial a \times 1$ (passes through ReLU since $z > 0$) | −3 |
| $\partial L / \partial w$ | $\partial L/\partial z \times x$ (multiplication: multiply by the other input $x = 2$) | −6 |
| $\partial L / \partial b$ | $\partial L/\partial z \times 1$ (addition: unchanged) | −3 |

**Update** (learning rate $\eta = 0.1$): $w = 0.5 - 0.1 \times (-6) = 1.1$, $b = 0.5 - 0.1 \times (-3) = 0.8$.

Running the forward pass again gives $z = 1.1 \times 2 + 0.8 = 3.0$ and $L = (3.0 - 3)^2 = 0$. We reached the target in one step (the numbers in this example were chosen to work out exactly; in practice it takes thousands of steps). A negative gradient means "increasing $w$ reduces the loss," which is why gradient descent increased $w$.

## 5. A Whole Layer: Writing It with Matrices

A real network handles hundreds of neurons at once, so the same computation is written with matrices. A handwritten digit (MNIST) is a 28×28 = 784-pixel image, and a network with 128 hidden units and 10 outputs (digits 0–9) looks like this. $B$ is the number of samples fed in at once (the batch size).

$$z_1 = XW_1 + b_1, \quad h = \mathrm{ReLU}(z_1), \quad z_2 = hW_2 + b_2, \quad p = \mathrm{softmax}(z_2)$$

| Value | Shape | Meaning |
|---|---|---|
| $X$ | $(B, 784)$ | Input images |
| $W_1$ | $(784, 128)$ | First-layer weights |
| $h$ | $(B, 128)$ | Hidden-layer output |
| $W_2$ | $(128, 10)$ | Second-layer weights |
| $z_2,\ p$ | $(B, 10)$ | Per-digit scores and probabilities |

![Forward and backward passes of a two-layer network. The forward pass goes from X through z1, h, and z2 to compute the loss L while storing intermediate values, and the backward pass starts from L and passes gradients through z2, h, and z1 in order to get the gradients of W2 and W1|full](/img/01M6EJDSNBMM4DKNDGF4PX1EYM/750cb9dacd03.webp)

Backpropagation goes backward from the loss, exactly as in section 4.

**① Softmax + cross-entropy.** Combining the two functions from Part 1 and differentiating gives something surprisingly simple. For one sample, $L = -\log p_y$ and $p_k = e^{z_k} / \sum_j e^{z_j}$, so $L = -z_y + \log \sum_j e^{z_j}$. Differentiating with respect to $z_k$ gives

$$\frac{\partial L}{\partial z_k} = -\mathbb{1}[k = y] + \frac{e^{z_k}}{\sum_j e^{z_j}} = p_k - \mathbb{1}[k = y]$$

That is, **the probabilities with 1 subtracted from the correct class only**. For the example in Part 1 (probabilities 0.659, 0.242, 0.099, correct class 0), the gradient is $(-0.341,\ 0.242,\ 0.099)$. It is a signal to raise the correct score and lower the rest. Since we use the batch average, we divide by $B$.

**② Second layer.** $z_2 = hW_2 + b_2$ is a multiplication and an addition. Following the rules from section 3, the gradient of $W_2$ multiplies by the other input $h$, and the gradient of $h$ multiplies by the other input $W_2$.

$$\frac{\partial L}{\partial W_2} = h^\top \frac{\partial L}{\partial z_2}, \qquad \frac{\partial L}{\partial h} = \frac{\partial L}{\partial z_2} W_2^\top$$

If you are unsure where the transpose ($\top$) goes, **just match the shapes.** $\partial L / \partial W_2$ must have the same shape as $W_2$, $(128, 10)$. The only ingredients are $h$ $(B, 128)$ and $\partial L / \partial z_2$ $(B, 10)$, so $(128, B) \times (B, 10)$, that is, $h^\top \cdot \partial L/\partial z_2$, is the only option. This product automatically sums the gradients from each of the $B$ samples in the batch.

**③ ReLU.** Entries that were negative in the forward pass were blocked at 0, so their gradient is 0 too ($\odot$ is element-wise multiplication).

$$\frac{\partial L}{\partial z_1} = \frac{\partial L}{\partial h} \odot \mathbb{1}[z_1 > 0]$$

**④ First layer.** Same form as ②. We do not need the gradient of the input $X$, so we skip it.

$$\frac{\partial L}{\partial W_1} = X^\top \frac{\partial L}{\partial z_1}$$

$X$, $z_1$, and $h$ used in backpropagation are all **values computed in the forward pass**. It is the same as needing $u$ for $dy/du = 2u$ in section 2. So during training, the forward pass's intermediate values have to stay in memory until backpropagation finishes. That is why training uses much more memory than inference alone.

## 6. Implementing It from Scratch in NumPy

I translated the equations from section 5 directly into code. No framework like PyTorch, just NumPy.

```python
import numpy as np

def forward(p, X):
    z1 = X @ p['W1'] + p['b1']          # (B,128)
    h = np.maximum(z1, 0)               # ReLU
    z2 = h @ p['W2'] + p['b2']          # (B,10) scores
    z2 = z2 - z2.max(1, keepdims=True)  # subtract the largest score so exp doesn't overflow (same result)
    prob = np.exp(z2) / np.exp(z2).sum(1, keepdims=True)
    return prob, (X, z1, h)             # keep intermediate values for backprop

def backward(p, prob, cache, y):
    X, z1, h = cache
    B = len(y)
    dz2 = prob.copy(); dz2[np.arange(B), y] -= 1; dz2 /= B   # ① subtract 1 from the correct class only
    g = {'W2': h.T @ dz2, 'b2': dz2.sum(0)}                   # ② second layer
    dh = dz2 @ p['W2'].T
    dz1 = dh * (z1 > 0)                                       # ③ ReLU: entries that were negative get 0
    g['W1'] = X.T @ dz1; g['b1'] = dz1.sum(0)                 # ④ first layer
    return g

# minibatch gradient descent: batch 64, learning rate 0.1, 10 epochs (batch splitting abbreviated)
for epoch in range(10):
    for idx in batches(len(X_train), 64):
        prob, cache = forward(p, X_train[idx])
        for k, v in backward(p, prob, cache, y_train[idx]).items():
            p[k] -= 0.1 * v
```

The bias gradient is `.sum(0)` because of the last rule in section 3. The same bias was added to every sample in the batch (used in several places), so we sum the gradients returning from each sample. The weights start from He initialization ($\mathcal{N}(0, 2/n_{in})$); why we start this way is covered in the next post. The full code (including data loading and measurement) is about 70 lines.

### Verifying the Backpropagation First

Backprop code still trains "to some degree" even when it is wrong, so bugs are easy to miss. That is why we use the numerical differentiation from section 1 as a check. On a small model (20-8-3, batch 5), comparing numerical and backprop gradients for every weight gave a largest relative error of $2.4 \times 10^{-8}$. That means the two are effectively identical. Only after this check did I run training.

### Training Results

![Training loss and test accuracy per epoch. The loss drops from 0.37 to 0.056, and test accuracy rises from 93.2% to 97.7%|xl](/img/01M6EJDSNBMM4DKNDGF4PX1EYM/d4ae9025b809.webp)

| Epoch | 1 | 2 | 5 | 10 |
|---|---|---|---|---|
| Training loss | 0.365 | 0.197 | 0.101 | 0.056 |
| Test accuracy | 93.2% | 95.2% | 97.1% | **97.7%** |

Ten passes over the 60,000 training images took 1.6 seconds in total (including test evaluation after each epoch, about 0.16 seconds per epoch). The hardware is the CPU of a MacBook Pro M1 Max (NumPy 2.5). Accuracy already passes 93% after the first epoch, and after that the loss falls slowly while accuracy creeps up. The model got 9,770 of the 10,000 test images, which were not used for training, right. A network with one hidden layer stalls around 98%, and what breaks through that wall is the CNN, covered from Part 4 onward.

## 7. Why Train on Minibatches Instead of One Image at a Time

In Part 1, we said the data is grouped into batches of 64 and each batch takes one step (minibatches). There are two reasons for grouping. One is gradient quality. The gradient from a single image swings around depending on that one image, while the average over several images is closer to the direction of the whole dataset. The other is speed. Using the same code and changing only the batch size, I measured the time to process (forward + backward) one sample.

![Forward + backward time per sample by batch size. 174 microseconds at batch 1, 6.2 at 8, 1.5 at 64, 1.0 at 512, and 1.3 microseconds at 4096|xl](/img/01M6EJDSNBMM4DKNDGF4PX1EYM/92870f99be21.webp)

| Batch size | 1 | 8 | 64 | 512 | 4096 |
|---|---|---|---|---|---|
| Time per sample | 174µs | 6.2µs | 1.5µs | **1.0µs** | 1.3µs |

The amount of computation per sample is identical (about 410,000 multiply-adds), yet there is a 180x difference between batch 1 and 512. The reason lies in the matrix multiplication from section 5. With a batch of 1, the entire 407 KB weight matrix $W_1$ is read from memory, used for one sample, and discarded. With a batch of 512, the weights read once are reused for 512 samples. For a computer, fetching data is more expensive than computing, so the more times fetched data is reused, the faster it gets. Batch 4096 gets slightly slower again, probably because the input alone grows to 12.8 MB and no longer fits in the CPU cache (a guess).

That said, making the batch arbitrarily large reduces the number of steps per epoch, which can actually slow down training, and it uses more memory too. So in practice the batch size is chosen somewhere between tens and thousands. GPUs are devices specialized for processing large matrix multiplications like this all at once, so the gap becomes much larger with big batches.

## 8. Summary

- Backpropagation applies the chain rule backward from the loss to get the gradients of all weights for about the cost of one forward pass. It is 100,000x cheaper than numerical differentiation.
- Each operation only needs its local rule. Addition passes the gradient through unchanged, multiplication multiplies by the other input's value, and ReLU passes only the entries that passed in the forward pass.
- The gradient of softmax + cross-entropy is "probabilities − target (one-hot)." Where the transpose goes in a matrix gradient falls out automatically from matching shapes.
- Backpropagation reuses the forward pass's intermediate values. That is why training uses more memory than inference.
- A hand-built two-layer network reaches 97.7% on MNIST. Verify backprop code against numerical differentiation first.

**Next post:** This time I used He initialization and a learning rate of 0.1 without much explanation. Part 3 compares, with the same kind of hands-on experiments, what happens when initialization goes wrong, and how optimizers such as SGD, Momentum, and Adam, along with normalization, change training.

## References

- D. E. Rumelhart, G. E. Hinton, R. J. Williams, "[Learning representations by back-propagating errors](https://www.nature.com/articles/323533a0)", *Nature* 323, 1986.
- K. He et al., "[Delving Deep into Rectifiers](https://arxiv.org/abs/1502.01852)", ICCV 2015 (He initialization).
- I. Goodfellow, Y. Bengio, A. Courville, *[Deep Learning](https://www.deeplearningbook.org/)*, Section 6.5 "Back-Propagation and Other Differentiation Algorithms", MIT Press, 2016.
- Stanford CS231n, "[Backpropagation, Intuitions](https://cs231n.github.io/optimization-2/)" (backpropagation through computational graphs).
- Y. LeCun, C. Cortes, C. J. C. Burges, [The MNIST database of handwritten digits](https://yann.lecun.com/exdb/mnist/).
