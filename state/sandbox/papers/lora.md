# LoRA: Low-Rank Adaptation of Large Language Models

Given the empirical advantage of LoRA, we hope to further explain the properties of the low-rank adaptation learned from downstream tasks.
Note that the low-rank structure not only lowers the hardware barrier to entry which allows us to run multiple experiments in parallel, but also gives better interpretability of how the update weights are correlated with the pre-trained weights.
We focus our study on GPT-3 175B, where we achieved the largest reduction of trainable parameters (up to 10,000×\times) without adversely affecting task performances.

We perform a sequence of empirical studies to answer the following questions:
1) Given a parameter budget constraint, *which subset of weight matrices* in a pre-trained Transformer should we adapt to maximize downstream performance?
2) Is the “optimal” adaptation matrix Δ​W\Delta W *really rank-deficient*? If so, what is a good rank to use in practice?
3) What is the connection between Δ​W\Delta W and WW? Does Δ​W\Delta W highly correlate with WW? How large is Δ​W\Delta W comparing to WW?

### 7.2 What is the Optimal Rank rr for LoRA?

We turn our attention to the effect of rank rr on model performance.
We adapt {Wq,Wv}\{W\_{q},W\_{v}\}, {Wq,Wk,Wv,Wc}\{W\_{q},W\_{k},W\_{v},W\_{c}\}, and just WqW\_{q} for a comparison.

|  |  |  |  |  |  |  |
| --- | --- | --- | --- | --- | --- | --- |
|  | Weight Type | r=1r=1 | r=2r=2 | r=4r=4 | r=8r=8 | r=64r=64 |
| WikiSQL(±0.5\pm 0.5%) | WqW\_{q} | 68.8 | 69.6 | 70.5 | 70.4 | 70.0 |
| Wq,WvW\_{q},W\_{v} | 73.4 | 73.3 | 73.7 | 73.8 | 73.5 |
|  | Wq,Wk,Wv,WoW\_{q},W\_{k},W\_{v},W\_{o} | 74.1 | 73.7 | 74.0 | 74.0 | 73.9 |
| MultiNLI (±0.1\pm 0.1%) | WqW\_{q} | 90.7 | 90.9 | 91.1 | 90.7 | 90.7 |
| Wq,WvW\_{q},W\_{v} | 91.3 | 91.4 | 91.3 | 91.6 | 91.4 |
| Wq,Wk,Wv,WoW\_{q},W\_{k},W\_{v},W\_{o} | 91.2 | 91.7 | 91.7 | 91.5 | 91.4 |

Table 6: Validation accuracy on WikiSQL and MultiNLI with different rank rr. To our surprise, a rank as small as one suffices for adapting both WqW\_{q} and WvW\_{v} on these datasets while training WqW\_{q} alone needs a larger rr. We conduct a similar experiment on GPT-2 in [Section H.2](https://arxiv.org/html/2106.09685v2#A8.SS2 "H.2 Effect of 𝑟 on GPT-2 ‣ Appendix H Additional Experiments on Low-Rank Matrices ‣ LoRA: Low-Rank Adaptation of Large Language Models").

[Table 6](https://arxiv.org/html/2106.09685v2#S7.T6 "Table 6 ‣ 7.2 What is the Optimal Rank 𝑟 for LoRA? ‣ 7 Understanding the Low-Rank Updates ‣ LoRA: Low-Rank Adaptation of Large Language Models") shows that, surprisingly, LoRA already performs competitively with a very small rr (more so for {Wq,Wv}\{W\_{q},W\_{v}\} than just WqW\_{q}).
This suggests the update matrix Δ​W\Delta W could have a very small “intrinsic rank”.
To further support this finding, we check the overlap of the subspaces learned by different choices of rr and by different random seeds. We argue that increasing rr does not cover a more meaningful subspace, which suggests that a low-rank adaptation matrix is sufficient.

Subspace similarity between different rr.  Given Ar=8A\_{r=8} and Ar=64A\_{r=64} which are the learned adaptation matrices with rank r=8r=8 and 6464 using the *same pre-trained model*, we perform singular value decomposition and obtain the right-singular unitary matrices UAr=8U\_{A\_{r=8}} and UAr=64U\_{A\_{r=64}}.
We hope to answer: how much of the subspace spanned by the top ii singular vectors in UAr=8U\_{A\_{r=8}} (for 1≤i≤81\leq i\leq 8) is contained in the subspace spanned by top jj singular vectors of UAr=64U\_{A\_{r=64}} (for 1≤j≤641\leq j\leq 64)?
We measure this quantity with a normalized subspace similarity based on the Grassmann distance (See [Appendix G](https://arxiv.org/html/2106.09685v2#A7 "Appendix G Measuring Similarity Between Subspaces ‣ LoRA: Low-Rank Adaptation of Large Language Models") for a more formal discussion)

|  |  |  |  |
| --- | --- | --- | --- |
|  | ϕ​(Ar=8,Ar=64,i,j)=‖UAr=8i⊤​UAr=64j‖F2min⁡(i,j)∈[0,1]\phi(A\_{r=8},A\_{r=64},i,j)=\frac{||U\_{A\_{r=8}}^{i\top}U\_{A\_{r=64}}^{j}||\_{F}^{2}}{\min(i,j)}\in[0,1] |  | (4) |

where UAr=8iU\_{A\_{r=8}}^{i} represents the columns of UAr=8U\_{A\_{r=8}} corresponding to the top-ii singular vectors.

ϕ​(⋅)\phi(\cdot) has a range of [0,1][0,1], where 11 represents a complete overlap of subspaces and 0 a complete separation.
See [Figure 3](https://arxiv.org/html/2106.09685v2#S7.F3 "Figure 3 ‣ 7.2 What is the Optimal Rank 𝑟 for LoRA? ‣ 7 Understanding the Low-Rank Updates ‣ LoRA: Low-Rank Adaptation of Large Language Models") for how ϕ\phi changes as we vary ii and jj.
We only look at the 48th layer (out of 96) due to space constraint, but the conclusion holds for other layers as well, as shown in [Section H.1](https://arxiv.org/html/2106.09685v2#A8.SS1 "H.1 Correlation between LoRA Modules ‣ Appendix H Additional Experiments on Low-Rank Matrices ‣ LoRA: Low-Rank Adaptation of Large Language Models").

Figure 3: Subspace similarity between column vectors of Ar=8A\_{r=8} and Ar=64A\_{r=64} for both Δ​Wq\Delta W\_{q} and Δ​Wv\Delta W\_{v}. The third and the fourth figures zoom in on the lower-left triangle in the first two figures. The top directions in r=8r=8 are included in r=64r=64, and vice versa.

We make an *important observation* from [Figure 3](https://arxiv.org/html/2106.09685v2#S7.F3 "Figure 3 ‣ 7.2 What is the Optimal Rank 𝑟 for LoRA? ‣ 7 Understanding the Low-Rank Updates ‣ LoRA: Low-Rank Adaptation of Large Language Models").

Directions corresponding to the top singular vector overlap significantly between Ar=8A\_{r=8} and Ar=64A\_{r=64}, while others do not. Specifically, Δ​Wv\Delta W\_{v} (resp. Δ​Wq\Delta W\_{q}) of Ar=8A\_{r=8} and Δ​Wv\Delta W\_{v} (resp. Δ​Wq\Delta W\_{q}) of Ar=64A\_{r=64} share a subspace of dimension 1 with normalized similarity >0.5>0.5, providing an explanation of why r=1r=1 performs quite well in our downstream tasks for GPT-3.

Since both Ar=8A\_{r=8} and Ar=64A\_{r=64} are learned using the same pre-trained model, [Figure 3](https://arxiv.org/html/2106.09685v2#S7.F3 "Figure 3 ‣ 7.2 What is the Optimal Rank 𝑟 for LoRA? ‣ 7 Understanding the Low-Rank Updates ‣ LoRA: Low-Rank Adaptation of Large Language Models") indicates that the top singular-vector directions of Ar=8A\_{r=8} and Ar=64A\_{r=64} are the most useful, while other directions potentially contain mostly random noises accumulated during training.
Hence, the adaptation matrix can indeed have a very low rank.

Figure 4: Left and Middle: Normalized subspace similarity between the column vectors of Ar=64A\_{r=64} from two random seeds, for both Δ​Wq\Delta W\_{q} and Δ​Wv\Delta W\_{v} in the 48-th layer. Right: the same heat-map between the column vectors of two random Gaussian matrices. See [Section H.1](https://arxiv.org/html/2106.09685v2#A8.SS1 "H.1 Correlation between LoRA Modules ‣ Appendix H Additional Experiments on Low-Rank Matrices ‣ LoRA: Low-Rank Adaptation of Large Language Models") for other layers.

Subspace similarity between different random seeds.  We further confirm this by plotting the normalized subspace similarity between two randomly seeded runs with r=64r=64, shown in [Figure 4](https://arxiv.org/html/2106.09685v2#S7.F4 "Figure 4 ‣ 7.2 What is the Optimal Rank 𝑟 for LoRA? ‣ 7 Understanding the Low-Rank Updates ‣ LoRA: Low-Rank Adaptation of Large Language Models").
Δ​Wq\Delta W\_{q} appears to have a higher “intrinsic rank” than Δ​Wv\Delta W\_{v}, since more common singular value directions are learned by both runs for Δ​Wq\Delta W\_{q}, which is in line with our empirical observation in [Table 6](https://arxiv.org/html/2106.09685v2#S7.T6 "Table 6 ‣ 7.2 What is the Optimal Rank 𝑟 for LoRA? ‣ 7 Understanding the Low-Rank Updates ‣ LoRA: Low-Rank Adaptation of Large Language Models").
As a comparison, we also plot two random Gaussian matrices, which do not share any common singular value directions with each other.

### 7.3 How Does the Adaptation Matrix Δ​W\Delta W Compare to WW?

We further investigate the relationship between Δ​W\Delta W and WW.
In particular, does Δ​W\Delta W highly correlate with WW? (Or mathematically, is Δ​W\Delta W mostly contained in the top singular directions of WW?) Also, how “large” is Δ​W\Delta W comparing to its corresponding directions in WW?
This can shed light on the underlying mechanism for adapting pre-trained language models.

To answer these questions, we project WW onto the rr-dimensional subspace of Δ​W\Delta W by computing U⊤​W​V⊤U^{\top}WV^{\top}, with UU/VV being the left/right singular-vector matrix of Δ​W\Delta W. Then, we compare the Frobenius norm between ‖U⊤​W​V⊤‖F\|U^{\top}WV^{\top}\|\_{F} and ‖W‖F\|W\|\_{F}.
As a comparison, we also compute ‖U⊤​W​V⊤‖F\|U^{\top}WV^{\top}\|\_{F} by replacing U,VU,V with the top rr singular vectors of WW or a random matrix.

|  |  |  |  |  |  |  |
| --- | --- | --- | --- | --- | --- | --- |
|  | r=4r=4 | | | r=64r=64 | | |
|  | Δ​Wq\Delta W\_{q} | WqW\_{q} | Random | Δ​Wq\Delta W\_{q} | WqW\_{q} | Random |
| ‖U⊤​Wq​V⊤‖F=||U^{\top}W\_{q}V^{\top}||\_{F}= | 0.32 | 21.67 | 0.02 | 1.90 | 37.71 | 0.33 |
| ‖Wq‖F=61.95||W\_{q}||\_{F}=61.95 | ‖Δ​Wq‖F=6.91||\Delta W\_{q}||\_{F}=6.91 | | | ‖Δ​Wq‖F=3.57||\Delta W\_{q}||\_{F}=3.57 | | |

Table 7: The Frobenius norm of U⊤​Wq​V⊤U^{\top}W\_{q}V^{\top} where UU and VV are the left/right top rr singular vector directions of either (1) Δ​Wq\Delta W\_{q}, (2) WqW\_{q}, or (3) a random matrix. The weight matrices are taken from the 48th layer of GPT-3.

We draw *several conclusions* from [Table 7](https://arxiv.org/html/2106.09685v2#S7.T7 "Table 7 ‣ 7.3 How Does the Adaptation Matrix Δ⁢𝑊 Compare to 𝑊? ‣ 7 Understanding the Low-Rank Updates ‣ LoRA: Low-Rank Adaptation of Large Language Models").
First, Δ​W\Delta W has a stronger correlation with WW compared to a random matrix, indicating that Δ​W\Delta W amplifies some features that are already in WW.
Second, instead of repeating the top singular directions of WW, Δ​W\Delta W only *amplifies directions that are not emphasized in WW*.
Third, the amplification factor is rather huge: 21.5≈6.91/0.3221.5\approx 6.91/0.32 for r=4r=4.
See [Section H.4](https://arxiv.org/html/2106.09685v2#A8.SS4 "H.4 Amplification Factor ‣ Appendix H Additional Experiments on Low-Rank Matrices ‣ LoRA: Low-Rank Adaptation of Large Language Models") for why r=64r=64 has a smaller amplification factor.
We also provide a visualization in [Section H.3](https://arxiv.org/html/2106.09685v2#A8.SS3 "H.3 Correlation between 𝑊 and Δ⁢𝑊 ‣ Appendix H Additional Experiments on Low-Rank Matrices ‣ LoRA: Low-Rank Adaptation of Large Language Models") for how the correlation changes as we include more top singular directions from WqW\_{q}.
This suggests that the low-rank adaptation matrix potentially *amplifies the important features for specific downstream tasks that were learned but not emphasized in the general pre-training model*.