# Negative Weight Mitigation via Cell Resampling

This repository contains the evaluation task for the MCFM GSoC project. It implements a cell resampling algorithm to locally eliminate negative weights in calculations while preserving the underlying physics distributions.

## Repository Structure

* **`gsoc.ipynb`**: The main Jupyter Notebook containing the data loading, Born projection, KD-Tree resampling algorithm, and verification plotting.
* **`virtual_events.csv`**: Toy dataset containing $N$-body virtual correction events with negative weights.
* **`real_events.csv`**: Toy dataset containing $(N+1)$-body real emission events with predominantly positive weights.
* **`verification_plot.png`**: The final output histograms demonstrating that the $p_T$ and rapidity ($y$) distributions are almost perfectly conserved after all negative weights are eliminated.

## Algorithm Structure

The solution is divided into three logical phases:

1.  **Born Projection:** The real emission events are mapped to the Born space using the provided projection formulas ($p_T = p_{T,real} + z_{gluon}$ and $y = y_{real}$). Both datasets are then concatenated into a single, unified list of events defined by $(p_T, y, weights)$.
2.  **Dynamic Cell Building (The KD-Tree Search):**
    To efficiently group events, the spatial coordinates are loaded into a `scipy.spatial.KDTree`. The algorithm iterates through the data to identify "seed" events (those with $w < 0$). For each seed, it queries the KD-Tree for the nearest unassigned neighbors, dynamically growing a "cell" until the sum of weights within the cell becomes positive ($\sum w \geq 0$).
3.  **Weight Redistribution:**
    Once a valid cell is formed, the weights of all events inside the cell are replaced using the formula:
    $$w_{new} = |w_{old}| \frac{\sum w}{\sum |w|}$$
    This guarantees that the local cross-section is conserved, but every individual event weight becomes strictly positive.

## Computational Complexity

A naive implementation of a nearest-neighbor search across $N$ events would require an $O(N^2)$ distance matrix, which scales poorly for large collider datasets. 

To optimize this, I utilized a **KD-Tree** spatial index. 
* **Building the tree:** $O(N \log N)$
* **Finding neighbors:** $O(\log N)$ per loop.
* **Overall Complexity:** The total time complexity of the resampling algorithm is essentially **$O(N \log N)$**, making it efficient and scalable.

## Discussion Question: The Scaling Factor

**Why is the scaling factor of 100 necessary?**
The coordinates $p_T$ and $y$ exist on vastly different numerical scales. Transverse momentum ($p_T$) is measured in GeV and typically ranges in the tens to hundreds, while rapidity ($y$) is a dimensionless angular measure typically falling between -5 and 5. We can see this by analysing the given `.csv` files. By scaling the rapidity difference by a factor of 100 (or effectively stretching the $y$-axis by $\sqrt{100} = 10$ before the Euclidean distance calculation), we ensure that the algorithm treats a small angular separation with the same geometric "weight" as a much larger separation in energy. This is essentially the same as normalizing or standardizing the data features so that variables with vastly different numerical ranges do not disproportionately dominate a distance-based clustering algorithm.

**What happens if standard Euclidean distance is used?**
Without scaling, the $p_T$ differences would completely dominate the distance metric. The algorithm would group events that are close in $p_T$ but wildly separated in rapidity. Having previously worked on unsupervised anomaly detection for a 3.8 TeV W' boson hunt - where I had to carefully handle mass sculpting and decorrelate kinematic variables to prevent background bias - I have seen how unscaled phase-space features can wash out physical realities. Using standard Euclidean distance here would artificially stretch the cells along the $y$-axis, distorting the angular distributions and ruining the physical accuracy of the simulation.

**How does the limit of infinite generated events affect this choice?**
In the theoretical limit of infinite generated events, the phase space becomes infinitely dense. Thus, the distance to the nearest neighbor approaches zero, and the "cells" become infinitesimally small. In this limit, the choice of metric matters less, as the grouped events are essentially identical in both $p_T$ and $y$. However in any practical finite simulation, an unscaled metric will force asymmetric cell growth, introducing local biases long before the infinite limit is reached.
