# Recurrent Neural Network (RNN) Introduction

## What's an RNN?
RNN stands for **recurrent neural network**. It is one of the classic architectures for modeling sequences.

RNN is simple, but learning it can help us understand the foundations, such as encoding, loss functions, cross-entropy, and others.

This note explains the idea of the RNN, how it works, and how it is implemented.


## The Idea
An RNN can be used to generate/predict new **words** based on previous text. For example, if we feed "The big brown bear scares the children with its" into an RNN, we might expect it to generate "roar" as the next word.

Text generation can be considered as a conditional distribution: given a prefix, the model assigns probabilities to the next word.

`P(w_i | w_{i-1}, w_{i-2}, ..., w_0)`

The same prefix can lead to different valid next words. For "The big brown bear scares the children with its", the next word could be "roar", "paw", and so on. So the RNN calculates the probability of all the possible worlds.

The text can have an arbitrary length, but a model uses fixed-size vector functions, so we have to fix this
conflict.

One of the simplest approaches is n-grams, which assume that the probability of a token depends only on a small fixed window of preceding tokens.

`P(w_i | w_{i-1}, w_{i-2}, ..., w_0) ≈ P(w_i | w_{i-1}, w_{i-2}, ..., w_{i-n+1}).`

This model's limitation is obvious: a word can only depend on a limited number of previous words.

For example, consider the sentence, "The big brown bear scares the children with its roar". In the sentence, the word 'bear' is strongly determined by the previous two words, but in order to predict the word 'roar', you would have to look back at least six words to know that you are talking about a bear.

RNNs address this by compressing previous tokens into a **context (hidden state, a vector)**, and predicting the next word from that context.

`P(word_i | context_{i-1})`
`context_i = f(word_i, context_{i-1})`

So the work becomes learning the function \(f\) and the probability model \(P\).


## How It Works
From the previous section, we can represent the history using a vector (the hidden state), so the conditional probability becomes `P(word_i | context_{i-1})`. Now we need to express those in math and how to find the proper parameters.

First, assume we have a vocabulary; all words come from it. When we use an RNN to predict the next word, it outputs a probability distribution over the vocabulary.

For simplicity, suppose our vocabulary has three words: `a`, `b`, `c`. The output is a probability distribution over these three words, e.g. `[0.0, 0.7, 0.3]`.

We compute it as:

`y_t = W_hy h_t + b_y`
`p_t = softmax(y_t)`

Here, `h_t` is the context (hidden state), `y_t` is a vector of unnormalized scores (logits), and `softmax` normalizes these scores into probabilities.

The context is updated by:

`h_t = tanh(W_xh x_t + W_hh h_{t-1} + b_h)`

This combines the current input `x_t` with the previous context `h_{t-1}` through linear transformations, then applies a nonlinearity function to produce the new context.

After defining these expressions, we need to learn the parameters (𝑊ℎ𝑦, 𝑏𝑦, 𝑊𝑥ℎ, 𝑊ℎℎ, 𝑏ℎ).

The idea is simple: we start with an initial guess for the parameters and apply the model to the training data to see how well it predicts the correct next token. If the prediction is wrong, we adjust the parameters slightly and repeat this process until the model is good enough.

We do this using a loss function and gradients. The loss function measures the difference between the true answer and the model’s prediction. The gradient is the derivative of the loss with respect to the parameters, indicating how changing the parameters affects the loss. Using these gradients, we update the parameters to minimize the loss.

In RNN language models, the loss function is usually cross-entropy.

## A Ruby Implementation

For simplicity, we build a character-level RNN in Ruby that can generate characters based on a given text. You can find the code in this repo https://github.com/yfractal/rnn-rb.

The RNN has two major functions: `train` and `generate`. The `train` function trains the model on the given text, and `generate` produces new text from a prefix. Let's focus on `train` first; once `train` makes sense, `generate` becomes straightforward.

For training, we want to learn parameters that work well (𝑊ℎ𝑦, 𝑏𝑦, 𝑊𝑥ℎ, 𝑊ℎℎ, 𝑏ℎ). We start from random values, run the model on training data, measure how well it predicts, and then adjust the parameters to reduce the loss.

Before training, we need to convert the text into vectors. Since our RNN is character-based, we list the unique characters in the text and set the vector dimension to the number of unique characters. For each character's vector, we set one element to 1 and all others to 0. This encoding method is one-hot encoding.

For example, if the text contains only three characters, "a", "b", "c", they can be represented as `[1, 0, 0]`, `[0, 1, 0]`, and `[0, 0, 1]`.

During training, we take a sequence from the training data. We do two passes: a forward pass and a backward pass.

The forward pass collects loss: we compute the current context (`h`), compute the probability distribution of the next character, and then compare it with the true next character to get loss.

```ruby
x = @vectors[p + t]
h = Numo::NMath.tanh(@wxh.dot(x) + @whh.dot(h) + @bh)
y = @why.dot(h) + @by
prob = softmax(y)

loss += cross_entropy(prob, @vectors[p + t + 1])
```

The backward pass computes gradients and determines how to update the parameters.

This is done by applying the chain rule to compute derivatives.

For example:

`L_t = -log(p_t[target])`
`∂L_t/∂y_t = p_t - y_true`
`y_t = W_hy h_t + b_y`
`∂L_t/∂b_y = ∂L_t/∂y_t`

After we have the gradients, we update the parameters to reduce the loss:

`@why -= @learning_rate * dWhy`

`generate` is simple: we use the learned parameters to compute the context and the probability of the next character, sample (or pick) a character, then feed it back in to update the context and continue generating.

## Summary
RNNs are simple neural networks that are easy to implement and understand. But it shares the same foundation as others, such as transformers.

After getting hands-on experience with RNNs, learning other, more complex models often becomes easier.

This article explained the idea behind RNNs, how they work, and how they can be implemented. Hope it can help you have a better understanding of RNN and other sequence models.
