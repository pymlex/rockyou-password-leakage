# RockYou Password Analysis

## Overview

We analyse password structures in a RockYou leak sample and visualizes them as a similarity graph. The analysed password list `rockyou_0.02_test.txt`—which is freely distributed as an artifact for the PassLLM paper with all ethical considerations addressed—contains 100,000 passwords, one per line. After deduplication, 82,771 unique passwords remain while preserving the order of first occurrence. Here is the length distribution of the passwords:

<img width="580" height="432" alt="output_13_1" src="https://github.com/user-attachments/assets/29fb2697-b16c-478c-8e9e-3a7f44d1abe3" />

The goal is to study how passwords cluster into families of close variants. The graph is built from approximate candidate retrieval with MinHash + LSH and exact filtering with Levenshtein distance to simplify the computations.

## Similarity model

Each password is converted into character 3-grams and represented by a MinHash signature with 128 permutations. LSH is used as a fast candidate filter, and exact Levenshtein distance is computed only for the candidate pairs that survive the first stage. 

The edit distance is defined as

$$d_{\text{Lev}}(s,t)$$

the minimum number of single-character insertions, deletions, and substitutions needed to transform one string into another. For passwords, this captures the common edits people actually make: digits added at the end, small typos, symbol substitutions, dates, and name variants.

## Threshold search

We build a grid over `threshold_lsh` and `threshold_levenshtein`. The LSH threshold is swept from `0.30` to `0.90` with step `0.02`, and the Levenshtein threshold is checked from `1` to `6`. For every LSH value, candidate pairs are collected once and their exact distances are reused for the full Levenshtein sweep. The graph connectivity is tracked with DSU. 

<img width="800" height="500" alt="newplot (3)" src="https://github.com/user-attachments/assets/73329bc8-1d90-4192-9f3a-212031d86944" />

The candidate-pair curve drops very quickly as `threshold_lsh` increases. It can be explained as the stricter LSH settings prune the search much harder, so the number of exact comparisons collapses fast. On the log scale the trend is close to linear, which means the dependence is roughly exponential in practice. 

## Graph construction

The graph is undirected. A node is a unique password, and an edge is added when the Levenshtein distance is small enough. The threshold controls how aggressively close variants are merged into the same component. Low values keep the graph fragmented; higher values connect more variants but also risk merging unrelated strings.

DSU is used to track connected components while edges are added in distance order. The structure keeps the current number of components and the size of the largest component without rebuilding the graph from scratch every time. That makes the threshold sweep much cheaper.

## Results

The final graph was built at `threshold_lsh = 0.40` and `threshold_levenshtein = 3`. At that point the pipeline produced 490,544 candidate pairs, 190,003 edges, 37,138 connected components, and a largest connected component of size 39,885.

The 3D surfaces show the same trade-off from different angles. The edge-count surface grows as the Levenshtein threshold increases and as the LSH filter becomes less strict, which is the expected behavior for a similarity graph:

<img width="800" height="500" alt="newplot (2)" src="https://github.com/user-attachments/assets/7a5d7917-dd39-4246-a501-2b0352d3ba98" />

The connected-components surface moves in the opposite direction: once the thresholds become looser, small components begin to merge and the graph becomes less fragmented:

<img width="800" height="500" alt="newplot" src="https://github.com/user-attachments/assets/56ed1bea-0e3a-4498-90e9-b27e79f93ec0" />

The largest-component surface makes the phase transition even more visible. In the middle of the parameter range the graph still preserves local structure, while at the looser end the largest component starts to dominate and the graph becomes too dense for fine-grained analysis:

<img width="800" height="500" alt="newplot (1)" src="https://github.com/user-attachments/assets/98bc0e40-74c8-4890-aa09-12c9e93679d9" />

## Visualizations of components

The 2D subgraph plots show that password families are not random. They cluster around simple roots like `123456` and short transformations, which is exactly what one expects from human-created passwords and highlight the perspective of usage of the `123456` password family for trawling attacks. These are the graphs ranging from 10 to 1 vertices:

<img width="3906" height="4928" alt="output_30_0" src="https://github.com/user-attachments/assets/8dc4e3e4-ba60-4728-a2d6-ad6b4648f43c" />

And bigger subsets from the main component of the graph:

<img width="3265" height="4928" alt="output_32_0" src="https://github.com/user-attachments/assets/f089c7f8-dbf8-4df7-9b2b-a37186171d58" />

## Repository contents

`rockyou_0.02_test.txt` is the password sample used for graph analysis.

`rockyou_passwords_levenstein_graph.ipynb` contains deduplication, MinHash signature creation, LSH filtering, exact Levenshtein filtering, DSU-based connectivity tracking, and graph visualization.

## Takeaway

The RockYou leak has strong local structure. Passwords are grouped into families built from repeated roots, suffixes, dates, and small edits. MinHash and LSH are enough to find the candidate space quickly, and Levenshtein distance then makes the structure explicit. This repository turns that structure into a graph and shows how the choice of thresholds changes the graph from sparse and fragmented to dense and overconnected.

[1]: https://www.usenix.org/conference/usenixsecurity25/presentation/zou-yunkai?utm_source=chatgpt.com "Password Guessing Using Large Language Models"
[2]: https://arxiv.org/html/2604.12601v1?utm_source=chatgpt.com "LLM-Guided Prompt Evolution for Password Guessing ..."
