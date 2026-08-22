---
title: "Pytorch Basics"
date: "2026-05-24"
tags: ["pytorch", "deep-learning", "python"]
---

# 1. Creating tensors

- Random: `torch.rand(shape)` (uniform in [0,1)), `torch.randn(shape)` (normal).
- `torch.Tensor(2, 3)` uninitialized tensor (random garbage values).
- Zeros/ones/full: `torch.zeros(2,3)`, `torch.ones(2,3)`, `torch.full((2,3), 5)`
- From Python list / NumPy: `torch.tensor([[1,2],[3,4]])`, `torch.from_numpy(numpy_array)`
  → dtype follows source NumPy dtype (e.g., `Float64` → `torch.DoubleTensor`)
- Type-casting: `x.float()`, `x.long()`, or `torch.tensor(..., dtype=torch.int64)`
- In-place ops: methods with `_` (e.g., `x.fill_(5)`) modify the tensor in-place.

---

# 2. Inspecting tensors

- `x.shape` or `x.size()`
- `x.dtype`, `x.device`, `x.numel()` (num elements)
- `x.item()` -> Python scalar (only for single-element tensors)

---

# 3. Reshape / dimensions

- `x.view(new_shape)` (works if contiguous), `x.reshape(new_shape)` (safer)
- `x.unsqueeze(dim)` — add size-1 dim
- `x.squeeze(dim)` — remove size-1 dim
- `x.transpose(0,1)` or `x.t()` for 2D
- `x.permute(...)` reorder dims
- `x.contiguous()` to get contiguous memory before `.view()`

---

# 4. Indexing / slicing / advanced indexing

- Basic slicing: `x[:, :2]`, `x[0,1]`
- Fancy indexing: `torch.index_select(x, dim, idx_longtensor)`
- Pair indexing: `x[row_idx, col_idx]` where both are LongTensors
- `torch.nonzero(x)` or `x.nonzero()` for indices of nonzero elements

---

# 5. Concatenate / stack

- `torch.cat([a,b], dim=0)` — concatenate along an existing dimension
- `torch.stack([a,b], dim=0)` — add new dimension and stack

---

# 6. Broadcasting / arithmetic

- Usual ops: `+`, `-`, `*`, `/`, or functional: `torch.add`, `torch.mul`, ...
- Matrix multiply: `torch.mm(A,B)` (2D), `torch.matmul` (broadcast-aware)
- Batch matrix multiply: `torch.bmm(a, b)` for 3D `a` and `b`
- Aggregations: `x.sum(dim=...)`, `x.mean(dim=...)`, `x.max(dim=...)`

---

# 7. Linear algebra helpers

- `torch.mm(A,B)` — matrix multiply (2D)
- `torch.bmm(A,B)` — batch matrix multiply (3D)
- `torch.transpose`, `torch.inverse` (when square), `torch.trace`, `torch.eig` etc.

---

# 8. Random seeds

```Python
torch.manual_seed(42)
if torch.cuda.is_available(): torch.cuda.manual_seed_all(42)
```

---

# 9. Autograd / gradients (the core)

- Enable tracking: `x = torch.ones(2,2, requires_grad=True)`
- Forward: compute loss scalar `loss = some_fn(x)`
- Backward: `loss.backward()` computes gradients; check `x.grad`
- Zero grads before optimizer step: `optimizer.zero_grad()` or `model.zero_grad()`
- Inference / disable grad: use `with torch.no_grad():` or `torch.set_grad_enabled(False)`
- Detach from graph: `y = x.detach()` or `x.detach().cpu().numpy()`

---

# 10. Common neural-net building blocks

- Layers: `torch.nn.Linear`, `torch.nn.Conv2d`, `torch.nn.Embedding`, `torch.nn.LSTM`, ...
- Losses: `torch.nn.CrossEntropyLoss()`, `torch.nn.MSELoss()`, `torch.nn.BCEWithLogitsLoss()`
- Activation functions: `torch.nn.ReLU()`, `torch.nn.Sigmoid()`, `torch.nn.Softmax(dim=1)`, or functional `F.relu`
- Example simple module:

```Python
import torch.nn as nn
class MLP(nn.Module):
    def __init__(self, in_dim, hidden, out_dim):
        super().__init__()
        self.net = nn.Sequential(
            nn.Linear(in_dim, hidden),
            nn.ReLU(),
            nn.Linear(hidden, out_dim)
        )
    def forward(self, x):
        return self.net(x)
```

---

# 11. Training loop (minimal pattern)

```Python
model.train()
for epoch in range(epochs):
    for xb, yb in dataloader:
        xb, yb = xb.to(device), yb.to(device)
        preds = model(xb)
        loss = criterion(preds, yb)
        optimizer.zero_grad()
        loss.backward()
        optimizer.step()
```

- For evaluation:

```Python
model.eval()
with torch.no_grad():
    for xb, yb in val_loader:
        preds = model(xb)
        # compute metrics
```

---

# 12. Optimizers & schedulers

- Common optimizers: `torch.optim.SGD(model.parameters(), lr=...)`, `torch.optim.Adam(...)`
- Zero grads: `optimizer.zero_grad()`
- Step: `optimizer.step()`
- LR schedulers: `torch.optim.lr_scheduler.StepLR`, `ReduceLROnPlateau`, etc.

---

# 13. Save & load model

- Save weights: `torch.save(model.state_dict(), "m.pt")`
- Load:

```Python
model = MyModel(...)
model.load_state_dict(torch.load("m.pt", map_location=device))
model.to(device)
```

- Save whole model (less recommended): `torch.save(model, "full_model.pt")` and `torch.load(...)`

---

# 14. Data pipeline

- Implement `torch.utils.data.Dataset` (implement `__len__` and `__getitem__`)
- Wrap in `DataLoader(dataset, batch_size=..., shuffle=True, num_workers=4)`
- Use `torchvision.transforms` for images or `torchtext` / custom transforms for text.
- For reproducibility, be careful with `num_workers>0` & random seeds.

---

# 15. Useful tensor utilities

- `torch.arange(start, end)` — like numpy `arange`
- `torch.linspace(start, end, steps)`
- `torch.where(cond, x, y)` — elementwise condition
- `torch.topk(x, k)`, `torch.argmax(x, dim)`, `torch.argsort`
- `x.view(-1)`, `x.flatten()`
- `x.expand(...)` vs `x.repeat(...)` (repeat creates new memory, expand is view-like)
- `x.clone()` to copy
- `x.cpu().numpy()` to get NumPy array (after `detach()` if requires_grad)

---

# 16. Memory / performance tips

- Prefer `torch.no_grad()` when evaluating (saves memory).
- Move whole batch to device once, not per-element.
- Avoid frequent `.cpu()` / `.numpy()` inside training loops (costly).
- Use `pin_memory=True` in DataLoader for faster CPU→GPU transfers (for GPU training).
- For multi-GPU training, consider `torch.nn.DataParallel` or `torch.nn.parallel.DistributedDataParallel`.

---

# 17. Common pitfalls & gotchas

- Mixing CPU and CUDA tensors → `RuntimeError`.
- Forgetting `optimizer.zero_grad()` → gradients accumulate.
- Using `.item()` on non-scalar tensors → error.
- In-place ops (e.g., `x += y` on variables that require grad) can break autograd sometimes; prefer out-of-place or be careful.
- Computing target-encoding / normalization *using whole dataset* causes data leakage — compute on training fold only.

---

# 18. Short example snippets

Create tensor, reshape, and sum:

```Python
x = torch.arange(6).view(2,3)   # [[0,1,2],[3,4,5]]
col_sum = x.sum(dim=0)          # [3,5,7]
row_sum = x.sum(dim=1)          # [3,12]
```

Gradients example:

```Python
x = torch.ones(2,2, requires_grad=True)
y = (x + 2) * (x + 5) + 3
z = y.mean()
z.backward()
print(x.grad)  # gradient of z wrt x
```

Batch matrix multiply:

```Python
a = torch.rand(3,4,5)
b = torch.rand(3,5,4)
c = torch.bmm(a,b)   # shape (3,4,4)
```
