---
title: "Visualizing LLM Tensors: From Tokens to Batch Matrices"
date: "2026-05-24"
tags: ["transformers", "llm", "deep-learning"]
---

This explanation of how individual text prompts are tokenized, transformed into dense mathematical matrices, padded for static batching, and routed inside a Transformer model.

---

### 1. Single Prompt: 1D Array to 3D Tensor

Let's trace a single prompt: **"The cat sat"**

#### Step A: Tokenization (1D Array)
The raw string is converted into a 1D list of integer token IDs using the tokenizer's vocabulary map:
```text
Tokens: [ 128, 567, 2432 ]
Shape:  (3,)
```

#### Step B: The Batch Dimension (2D Matrix)
Deep learning libraries (like PyTorch or JAX) require a batch dimension at the very front of the tensor. Even with a single prompt, we wrap it in a nested array:
```text
Batch Tensor: [
  [ 128, 567, 2432 ]
]
Shape: (1, 3) -> (Batch Size = 1, Sequence Length = 3)
```

#### Step C: Embedding Lookup (3D Tensor / Dense Matrix)
An embedding layer (lookup table) maps each integer token ID to a dense vector of size **D** (Hidden Dimension, e.g., **D = 8** for this visualization):

```text
Token ID       Embedding Vector (Size D = 8)
-------------------------------------------------------------------------------
[ 128  ] --->  [  0.12, -0.45,  0.89,  0.03,  0.77, -0.19,  0.05, -0.66 ] (Vector 1)
[ 567  ] --->  [ -0.98,  0.22, -0.11,  0.74,  0.33,  0.50, -0.88,  0.12 ] (Vector 2)
[ 2432 ] --->  [  0.55,  0.01,  0.67, -0.32, -0.09,  0.82,  0.41,  0.95 ] (Vector 3)
```

If we stack these vectors together, they form a beautiful **2D Matrix** of size **3 × 8**:

```text
          Col 1   Col 2   Col 3   Col 4   Col 5   Col 6   Col 7   Col 8
         +-------+-------+-------+-------+-------+-------+-------+-------+
Token 1  |  0.12 | -0.45 |  0.89 |  0.03 |  0.77 | -0.19 |  0.05 | -0.66 |
         +-------+-------+-------+-------+-------+-------+-------+-------+
Token 2  | -0.98 |  0.22 | -0.11 |  0.74 |  0.33 |  0.50 | -0.88 |  0.12 |
         +-------+-------+-------+-------+-------+-------+-------+-------+
Token 3  |  0.55 |  0.01 |  0.67 | -0.32 | -0.09 |  0.82 |  0.41 |  0.95 |
         +-------+-------+-------+-------+-------+-------+-------+-------+
```

Including the batch size of 1, the actual shape inside the model is `(1, 3, 8)`. Every transformer layer performs high-speed matrix multiplications directly against this **3 × 8** token matrix.

---

### 2. Static Batching: Multiple Inputs

Suppose two users submit prompts at the same time:
* **User 1**: `"The cat"` $\rightarrow$ `[128, 567]` (Length = 2)
* **User 2**: `"A dog barked loudly"` $\rightarrow$ `[45, 902, 1821, 3445]` (Length = 4)

Because a GPU requires a perfect rectangular grid of inputs to process items in parallel, we must pad the shorter sequences.

#### Step 1: Finding Max Length & Padding
The longest sequence has a length of 4. We pad User 1's sequence with a placeholder `<pad>` token (represented as `0`) to match length 4:

```text
User 1 (Padded): [ 128,  567,    0,    0 ]
User 2:          [  45,  902, 1821, 3445 ]
```

#### Step 2: Creating the 2D Batch Matrix
We stack the sequences vertically to form a single 2D matrix of shape `(Batch Size, Max Sequence Length)` $\rightarrow$ `(2, 4)`:

```text
                  Token 1    Token 2    Token 3    Token 4
                +----------+----------+----------+----------+
Row 1 (User 1)  |   128    |   567    |    0     |    0     |  <-- Padded tokens
                +----------+----------+----------+----------+
Row 2 (User 2)  |    45    |   902    |   1821   |   3445   |
                +----------+----------+----------+----------+
```

#### Step 3: Embedding to 3D Tensor
When this `(2, 4)` token matrix passes through the embedding layer, each slot is replaced by an 8-dimensional vector, resulting in a 3D tensor of shape `(2, 4, 8)`:

```text
                  |--- Token 1 ---| |--- Token 2 ---| |--- Token 3 ---| |--- Token 4 ---|
Row 1 (User 1)    [ e_128 vector  ] [ e_567 vector  ] [   e_0 vector  ] [   e_0 vector  ]
Row 2 (User 2)    [ e_45 vector   ] [ e_902 vector  ] [ e_1821 vector ] [ e_3445 vector ]
```

#### Step 4: The Attention Mask (Discarding Padding)
During Self-Attention, we multiply the Query (Q) and Key (K) matrices to calculate attention scores. For User 1, we do not want the model to pay attention to the padded `0` columns. 

We apply an **Attention Mask** of shape `(2, 4)`:

```text
                Token 1   Token 2   Token 3   Token 4
               +---------+---------+---------+---------+
User 1 Mask    |    1    |    1    |    0    |    0    |  <-- Zeros force attention away
               +---------+---------+---------+---------+
User 2 Mask    |    1    |    1    |    1    |    1    |
               +---------+---------+---------+---------+
```

The scores where the mask is `0` are set to negative infinity ($-\infty$), which makes them evaluate to exactly `0` when passed through the Softmax activation function.

---

### 3. Extracting the Next Token Prediction

After the model runs all forward passes, it yields output logits of shape `(Batch Size, Max Sequence Length, Vocab Size)`.

Suppose our vocabulary size is 5 (for simple visualization):

```text
Batch Output Logits Tensor (Shape: 2 x 4 x 5)

[ USER 1 LOGITS ]
                    "The"     "cat"     "sat"     "dog"     "barked"
                  +---------+---------+---------+---------+---------+
Token 1 (128)     |   0.1   |   0.8   |   0.0   |   0.1   |   0.0   |
                  +---------+---------+---------+---------+---------+
Token 2 (567)     |   0.0   |   0.1   |   0.7   |   0.1   |   0.1   | <-- EXTRACT (Next word after "cat")
                  +---------+---------+---------+---------+---------+
Token 3 (pad)     |   0.2   |   0.2   |   0.2   |   0.2   |   0.2   | <-- Ignored
                  +---------+---------+---------+---------+---------+
Token 4 (pad)     |   0.2   |   0.2   |   0.2   |   0.2   |   0.2   | <-- Ignored
                  +---------+---------+---------+---------+---------+

[ USER 2 LOGITS ]
                    "The"     "cat"     "sat"     "dog"     "barked"
                  +---------+---------+---------+---------+---------+
Token 1 (45)      |   0.0   |   0.1   |   0.0   |   0.9   |   0.0   |
                  +---------+---------+---------+---------+---------+
Token 2 (902)     |   0.1   |   0.1   |   0.1   |   0.1   |   0.6   |
                  +---------+---------+---------+---------+---------+
Token 3 (1821)    |   0.0   |   0.0   |   0.0   |   0.2   |   0.8   |
                  +---------+---------+---------+---------+---------+
Token 4 (3445)    |   0.7   |   0.1   |   0.1   |   0.0   |   0.1   | <-- EXTRACT (Next word after "loudly")
                  +---------+---------+---------+---------+---------+
```

#### Finding the Predictions
* For **User 1**, the last actual token was at index `1` ("cat"). We extract row index 1: `[0.0, 0.1, 0.7, 0.1, 0.1]`. The highest value is `0.7` under **"sat"**.
* For **User 2**, the last actual token was at index `3` ("loudly"). We extract row index 3: `[0.7, 0.1, 0.1, 0.0, 0.1]`. The highest value is `0.7` under **"The"**.
