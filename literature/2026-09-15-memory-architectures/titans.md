# Verification report: Google's neural memory line (Titans, MIRAS, Nested Learning / Hope)

Date of check: 2026-09-15. Method: every claim below was checked against primary sources (arXiv abstract pages and full HTML text, the NeurIPS proceedings and poster pages, Google Research blog posts, GitHub API metadata via `gh`, Hugging Face API/config.json, PyPI API). Figure values marked "read from plot" are approximate readings of the paper figures downloaded from the arXiv HTML build. Nothing here is taken from a project's self-description without being checked.

## 0. Verdict table

| # | Claim (ChatGPT, Sept 2026) | Verdict | Key evidence |
|---|---|---|---|
| 1 | Titans (arXiv 2501.00663, Behrouz et al.) introduced a neural long-term memory that learns at inference time; Google reports scaling beyond 2M tokens with much better long-range recall than transformer and recurrent baselines | Partly verified | Paper exists and says exactly that, but the only evidence past 16K is one fine-tuned BABILong curve with points at 1M and 10M; largest model 760M; no code or weights ever released. https://arxiv.org/abs/2501.00663 |
| 2 | MIRAS (2504.13173) generalizes Titans; Nested Learning and Hope (NeurIPS 2025, Google blog Nov 2025) introduce a Continuum Memory System with memories at different update rates and a model that modifies its own memory while operating | Verified (as the authors' claims) | All records, venue, dates and definitions check out. Hope is trained from scratch at 760M/1.3B; on hard NIAH variants at 16K it trails a plain Transformer badly. https://arxiv.org/abs/2504.13173 https://arxiv.org/abs/2512.24695 |
| 3 | Still architecture research; you cannot bolt Hope onto GPT-6/Qwen tomorrow | Verified, with one caveat | No official code or weights for Titans, MIRAS or Hope; all Google results are from-scratch training. Caveat: a single-author community project (TPTT) does graft a Titans-style gate onto Qwen2.5-1.5B via LoRA, but it was evaluated on 500 Alpaca samples with MMLU only. https://arxiv.org/abs/2506.17671 |

## 1. Claim 1: Titans

### 1.1 Record
- Title: "Titans: Learning to Memorize at Test Time". Authors: Ali Behrouz, Peilin Zhong, Vahab Mirrokni, Google Research. Submitted 31 Dec 2024, v1 only (https://arxiv.org/abs/2501.00663v2 returns 404). License CC BY 4.0. Source: https://arxiv.org/abs/2501.00663
- Peer-reviewed venue: NeurIPS 2025 poster, main conference. Source: https://neurips.cc/virtual/2025/poster/119639
- Google Research blog: "Titans + MIRAS: Helping AI have long-term memory", 4 Dec 2025, by Behrouz, Razaviyayn, Mirrokni. It repeats "the capability to scale effectively to context window sizes larger than 2 million tokens" and "Titans outperforms all baselines, including extremely large models like GPT-4". It links only the two arXiv papers; no code, weights or deployment mentioned. Source: https://research.google/blog/titans-miras-helping-ai-have-long-term-memory/

### 1.2 What was trained
- Model sizes 170M, 340M, 400M, 760M. "first three are trained on 15B tokens sampled from FineWeb-Edu dataset, the last one is trained on 30B tokens from the same dataset". Source: https://arxiv.org/html/2501.00663v1 (Section 5.1)
- Baselines for LM/reasoning: Transformer++, RetNet, GLA, Mamba, Mamba2, DeltaNet, TTT, Gated DeltaNet, HGRN2, Samba, Gated DeltaNet-H2.

### 1.3 Needle in a haystack (RULER S-NIAH), Table 2
- Lengths tested: 2K, 4K, 8K, 16K only. Quote: "evaluate Titans and baselines on sequences with length 2K, 4K, 8K, and 16K."
- At 16K (PK / N / W): Titans (MAC) 98.4 / 97.4 / 95.2; Titans (LMM) 96.2 / 80.2 / 80.6; TTT 88.4 / 4.4 / 0.0; Mamba2 5.4 / 0.0 / 0.0; DeltaNet 71.4 / 5.4 / 0.0. There is no Transformer row in this table. Source: https://arxiv.org/html/2501.00663v1 (Table 2)

### 1.4 BABILong, Figure 6 (values read from plot)
- Figure 6(a), few-shot, x-axis 1K to 1M: Titans (MAC) about 76% at 1K, 71% at 16K, 64% at 128K, 61% at 512K, 53% at 1M. GPT-4 about 81% at 1K falling to 36% at 128K; GPT4o-mini 43% and Llama3.1-8B 39% at 128K; Mamba2.8B, RWKV-6-7B and Gemma-9B are near 0 by 8K; RecurrentGemma-9B about 8% at 16K. No baseline is plotted past 128K. Figure file: https://arxiv.org/html/2501.00663v1/Figures/BABILong-few-shot.png
- Figure 6(b), fine-tuned, x-axis 1K to 10M: Titans (MAC)-FT about 99% through 128K, 95% at 512K, 94% at 1M, 69% at 10M. RMT-FT about 34% at 10M, Llama3.1-8B + RAG about 37% at 10M. Mamba-FT ends at 128K (about 92%). GPT-4 (36%), Qwen2.5-72B (58%), Llama3.1-70B (45%) end at 128K. Figure file: https://arxiv.org/html/2501.00663v1/Figures/BABILong-FT.png
- Paper text: "Baseline results are reported by (Kuratov et al., 2024)" and "we fine-tune the MAC variant of Titans to compare it with other fine-tuned baselines." Source: https://arxiv.org/html/2501.00663v1 (Section 5.4)

### 1.5 The 2M sentence and what actually backs it
- The claim appears three times: abstract ("They further can effectively scale to larger than 2M context window size with higher accuracy in needle-in-haystack tasks compared to baselines."), introduction ("contrary to Transformers, Titans scale to larger than 2M context window size") and conclusion ("Titans can scale to larger than 2M context window size with better accuracy than baselines"). Source: https://arxiv.org/abs/2501.00663 and https://arxiv.org/html/2501.00663v1
- Backing: no S-NIAH result beyond 16K. The only measurements past 128K are the fine-tuned BABILong curve (points at 512K, 1M, 10M) and the few-shot curve (to 1M). There is no data point at 2M; "larger than 2M" is the 1M to 10M segment of one fine-tuned curve. The transformer baselines (GPT-4, Qwen2.5-72B, Llama3.1-70B) stop at 128K; the only baselines at 10M are RMT-FT and Llama3.1-8B + RAG, both far below Titans.
- So: the paper says what ChatGPT says, but "successful scaling beyond 2M while maintaining much better long-range recall than transformer and recurrent baselines" rests on a 760M-or-smaller model, a single fine-tuned benchmark, baseline numbers copied from the benchmark authors, and no released code.

### 1.6 Code and weights
- Paper: "Titans are implemented in Pytorch and JAX and we intend to make the code we used to train and evaluate our models available soon." Source: https://arxiv.org/html/2501.00663v1 (Conclusion)
- GitHub search of the google-research, google-deepmind and google organizations for "titans", "nested learning" and "miras" via `gh search repos --owner ...` returned no repositories (only the unrelated google/tock-on-titan). No official code or weights exist as of 2026-09-15. Blog posts link no code: https://research.google/blog/titans-miras-helping-ai-have-long-term-memory/

### 1.7 Verdict: Partly verified
The paper, authors, id and the 2M sentence are real. The evidence for ">2M" is thin (one fine-tuned BABILong curve), the models are small (max 760M, 30B tokens), and nothing is reproducible from official code.

## 2. Claim 2: MIRAS, Nested Learning, Hope

### 2.1 MIRAS
- Title: "It's All Connected: A Journey Through Test-Time Memorization, Attentional Bias, Retention, and Online Optimization". Authors: Ali Behrouz, Meisam Razaviyayn, Peilin Zhong, Vahab Mirrokni. Submitted 17 Apr 2025, v1. Source: https://arxiv.org/abs/2504.13173
- Content: "we present Miras, a general framework to design deep learning architectures based on four choices of: (i) associative memory architecture, (ii) attentional bias objective, (iii) retention gate, and (iv) memory learning algorithm." Titans-LMM is described as a special instance of Miras (nonlinear MSE objective, local plus global retention, gradient descent with momentum). Three new models: Moneta, Yaad, Memora. So "MIRAS generalizes Titans" is verified. Source: https://arxiv.org/html/2504.13173v1
- Experiments: sizes 120M, 340M, 760M, 1.3B on 15B/15B/30B/100B tokens (FineWeb-Edu; C4 for scaling); training context 4096; S-NIAH at 1K to 8K only (Table 3). No code statement anywhere in the paper. Source: https://arxiv.org/html/2504.13173v1 (Section 6)

### 2.2 Nested Learning and Hope
- Title: "Nested Learning: The Illusion of Deep Learning Architectures". Authors: Ali Behrouz, Meisam Razaviyayn, Peilin Zhong, Vahab Mirrokni. arXiv submitted 31 Dec 2025, v1 only (v2 returns 404). arXiv comments: "A version of this work is published at Neural Information Processing Systems (NeurIPS) 2025". Source: https://arxiv.org/abs/2512.24695
- NeurIPS 2025 main-track proceedings entry: https://proceedings.neurips.cc/paper_files/paper/2025/hash/4309616aaed8e848009bc4a7ef73b493-Abstract-Conference.html ; poster page: https://neurips.cc/virtual/2025/poster/116123 ; author-hosted PDF: https://alibehrouz.com/files/NL.pdf
- Google Research blog: "Introducing Nested Learning: A new ML paradigm for continual learning", 7 Nov 2025, by Ali Behrouz (Student Researcher) and Vahab Mirrokni (VP and Google Fellow). Source: https://research.google/blog/introducing-nested-learning-a-new-ml-paradigm-for-continual-learning/
- Blog quotes: "The Nested Learning paradigm extends this concept into what we call a 'continuum memory system' (CMS), where memory is seen as a spectrum of modules, each updating at a different, specific frequency rate." and "Hope, however, is a self-modifying recurrent architecture that can take advantage of unbounded levels of in-context learning and also is augmented with CMS blocks to scale to larger context windows. It can essentially optimize its own memory through a self-referential process, creating an architecture with infinite, looped learning levels." The post links no code, weights or GitHub.

### 2.3 Exact definitions in the paper
- CMS (Section 7.1): "Continuum Memory System (CMS) is formalized as a chain of MLP blocks MLP^(f1), ..., MLP^(fk), each of which associated with a chunk size C^(l)" i.e. each block is updated at its own frequency; "higher-frequency neurons are responsible for fast adaption but store memories/knowledge for a short period of time, while lower frequency neurons are responsible for more persistent knowledge." Source: https://arxiv.org/html/2512.24695v1
- Self-modifying (Section 8.1): "we present self-modifying deep associative memory where the models generate their own values" and, from the abstract, "a sequence model that learns how to modify itself by learning its own update algorithm". Hope (Section 8.3): "A neural learning module that incorporates self-modifying Titans followed by Continuum Memory System." A second variant, "Hope-Attention", replaces the self-modifying Titans block with softmax global attention. Source: https://arxiv.org/html/2512.24695v1
- So "multiple memory systems at different update rates" and "modifies its own memory while operating" are accurate paraphrases of what the authors claim their design does.

### 2.4 What Hope actually demonstrates (numbers)
- Training: "We train models with about 760M and 1.3B parameters, trained with 30B and 100B tokens, respectively, from a mixture of FineWeb-Edu and long-context documents with a vocabulary size of 32K to train all the models from scratch." Long-context runs use "about 50B tokens". Source: https://arxiv.org/html/2512.24695v1 (Section 9)
- NIAH, Table 1, at 4K / 8K / 16K: S-NIAH-1 Hope 100/100/100 (Titans 100/100/100, Transformer 88.6/76.4/79.8). S-NIAH-2 Hope 99.2/88.4/78.2 vs Titans 99.6/84.6/75.4 vs Transformer 100/98.8/94.2. S-NIAH-3 Hope 73.2/46.2/24.8 vs Titans 74.2/42.8/21.2 vs Transformer 78.0/69.2/40.8. Multi-key NIAH at 16K: Transformer 61.4, Hope 14.8, Titans 8.2. Multi-query at 16K: Transformer 29.8, Hope 14.2. Multi-value at 16K: Transformer 21.5, Hope 11.4. Hope-Attention roughly matches the Transformer. Read: the attention-free Hope beats Titans, RWKV-7, Comba and DLA, but a plain Transformer is far better on the harder NIAH variants at 16K. Source: https://arxiv.org/html/2512.24695v1 (Table 1)
- BABILong, Figure 9 (fine-tuned setting, values read from plot; x-axis 0K to 10M): Hope about 99% through 512K, 97% at the next tick, 94% at 10M; Titans about 93% then 69% at 10M; ARMT about 77% at 10M; RMT about 34%; Llama3 + RAG about 37%; GPT-4, GPT4o-mini and Llama-8B are at 0 from 512K on. Paper text: "Among fine-tuned models, Titans, ARMT, and Hope show competitive results until 1M context length, but the performance of both Titans and ARMT drop fast after that point. Hope maintains its good performance even for 10M context length, mainly due to its CMS design. It is notable that the performance of all small models, including Hope, can drop significantly when used without fine-tuning." Figure file: https://arxiv.org/html/2512.24695v1/Images/BABILong.png
- Ablation (Table 3): Hope 12.24 ppl / 58.1 acc; w/o Delta Gradient Descent 13.41 / 56.5; w/o momentum 13.58 / 56.9; w/o weight decay 13.71 / 57.2; w/o CMS 13.04 / 57.3. Source: https://arxiv.org/html/2512.24695v1
- Other tasks named: continual learning (learning a new language, class-incremental text classification on CLINC/Banking/DBpedia, QA on a new corpus), MK-NIAH/LongHealth/QASPER for memory levels, ViT and LM optimizer experiments for the M3 optimizer.

### 2.5 Code and weights
- No code or release statement in the paper or the blog. No repository in google-research / google-deepmind / google orgs. The most careful community reproduction states: "No official code for HOPE has been released" and "No pretrained weights are distributed". Source: https://github.com/smallhours19/nested-learning-hope

### 2.6 Verdict: Verified as the authors' claims
Dates, venue, authorship and the CMS / self-modification descriptions are accurate. Scale is 760M to 1.3B, from scratch, and the strongest long-context number (94% at 10M BABILong) requires fine-tuning on the benchmark.

## 3. Claim 3: "still architecture research, not a bolt-on for GPT-6/Qwen"

### 3.1 Evidence for
- Every Google result above is from-scratch pretraining at 1.3B or less with no code or weights (Sections 1.6, 2.5). The Hope paper itself says small models "can drop significantly when used without fine-tuning" on BABILong. No paper by the authors adapts a pretrained Transformer into Hope.

### 3.2 Counter-evidence (community, Titans only)
- TPTT: "TPTT: Transforming Pretrained Transformers into Titans", Fabien Furfaro (single author, no Google affiliation), v1 21 Jun 2025, v2 31 Aug 2025. Injects linearized attention (LiZA) plus "Memory as Gate" into Hugging Face models with LoRA; tested on Llama-3.2-1B, Qwen2.5-1.5B, OLMo-1B, OpenELM-1.3B, Gemma-270m, OLMoE-1B-7B, Mistral-7B. Training: 500 Alpaca-cleaned samples, 5 epochs, 512-token sequences, Tesla T4. Evaluation: MMLU one-shot only; Titans-Llama-3.2-1B exact match 0.246 vs baseline 0.007 but partial exact match 0.265 vs 0.311; several models scored 0.000 exact match; no long-context or NIAH evaluation at all; authors state "More extensive empirical validation ... is necessary". Sources: https://arxiv.org/abs/2506.17671 and https://arxiv.org/html/2506.17671v2
- Repo: https://github.com/fabienfrfr/tptt (65 stars, Apache-2.0, last push 2026-06-07); PyPI tptt 0.12.1 uploaded 2025-09-27; example uses `Qwen/Qwen2.5-1.5B`. Checkpoints (Hugging Face, ffurfaro): Titans-Qwen2.5-1.5B, Titans-Llama-3.2-1B, Titans-OpenELM-1_1B, Titans-OLMo-1B-hf, Titans-v2-OLMoE-1B-7B-0924, Titans-v2-Mistral-7B-v0.3, e.g. https://huggingface.co/ffurfaro/Titans-Qwen2.5-1.5B (LoRA adapters plus LiZA injection, Alpaca fine-tune, Apache-2.0).
- ddidacus/llama-titans: wraps `meta-llama/Llama-3.2-1B` with `TitanModel.from_pretrained(..., gated=True/False)` (MAG / MAL). 25 stars, MIT, last push 2025-03-06, README marked "wip", no results. Source: https://github.com/ddidacus/llama-titans

### 3.3 Verdict: Verified, with the TPTT caveat
Nothing lets you attach Hope to an existing model. A Titans-style gate can be LoRA-grafted onto Qwen2.5-1.5B with TPTT, but there is no evidence it delivers Titans' long-context behaviour; it is a demo.

## 4. Code landscape (all metadata from the GitHub API on 2026-09-15)

| Repo | Stars / forks | License | Created / last push | What it is | Apple Silicon | Pretrained integration |
|---|---|---|---|---|---|---|
| Official Google (google-research, google-deepmind, google) | none found | n/a | n/a | No Titans, MIRAS, Nested Learning or Hope repo exists | n/a | n/a |
| https://github.com/lucidrains/titans-pytorch | 1,981 / 207 | MIT | 2025-01-08 / 2026-07-13 | "Unofficial implementation of Titans"; `NeuralMemory` and `MemoryAsContextTransformer` (MAC), `train_mac.py`; PyPI titans-pytorch 0.5.5 (2026-07-13, 167 releases) | Plain PyTorch, README examples use `.cuda()`, no MPS/Apple mention | No, trains from scratch |
| https://github.com/Aedelon/titans-pytorch-mlx | 29 / 6 | Apache-2.0 | 2026-01-15 / 2026-01-16 | PyTorch plus MLX ports of MAC/MAG/MAL/LMM with Metal kernels and PyTorch MPS backend, numerical parity tests, pretraining scripts streaming FineWeb-Edu | Yes, MLX requires macOS 13.5+ and M1 to M4 | No, pretraining from scratch, no weights |
| https://github.com/fabienfrfr/tptt | 65 / 2 | Apache-2.0 | 2025-05-09 / 2026-06-07 | LiZA plus Memory-as-Gate injection with LoRA into HF models | Not addressed (paper used T4 GPUs) | Yes: Qwen2.5-1.5B, Llama-3.2-1B and others; HF checkpoints exist |
| https://github.com/ddidacus/llama-titans | 25 / 2 | MIT | 2025-02-20 / 2025-03-06 | MAG/MAL wrapper around HF Llama-3.2-1B, "wip" | Not addressed | Yes (Llama), no results |
| https://github.com/kolejnyy/titans-lmm | 24 / 5 | none | 2025-04-09 / 2025-04-09 | Proof of concept | Not addressed | No |
| https://github.com/obekt/HOPE-nested-learning | 89 / 17 | none | 2025-12-01 / 2026-06-11 | Unofficial from-scratch Hope: delta-rule fast weights plus multi-frequency CMS tiers | README: "NVIDIA CUDA or Mac MPS supported" | No |
| https://github.com/smallhours19/nested-learning-hope | 0 / 0 | MIT | 2026-09-04 / 2026-09-04 | Careful unofficial reproduction at 161.5M and 0.3B on FineWeb-Edu; states no official code and no weights | Not addressed (2-GPU DDP) | No |
| https://github.com/aryateja2106/nested-learning | 12 / 1 | Apache-2.0 | 2025-12-07 / 2025-12-10 | Open-source implementation plus a business demo | Not addressed | No |
| https://github.com/rlaope/HOPE-tensorflow | 3 / 0 | MIT | 2026-05-19 / 2026-05-30 | TensorFlow port | Not addressed | No |
| WindOfNature/Nested-Learning | gone | n/a | n/a | Appears in search indexes; GitHub API returns 404 for both repo and user | n/a | n/a |

Adjacent non-Google artifacts, for calibration:
- Hugging Face org Test-Time-Training publishes ttt-linear / ttt-mlp checkpoints from 125M to 1.3B (Sun et al. 2024 TTT layers), research checkpoints only: https://huggingface.co/Test-Time-Training
- "End-to-End Test-Time Training for Long Context" (TTT-E2E), Tandon ... Yejin Choi, Yu Sun, v1 29 Dec 2025, 3B models, 128K and 2M contexts, code at https://github.com/test-time-training/e2e ; does not use or cite Titans. Source: https://arxiv.org/abs/2512.23675 ; NVIDIA blog 9 Jan 2026 claims no production deployment: https://developer.nvidia.com/blog/reimagining-llm-memory-using-context-as-training-data-unlocks-models-that-learn-at-test-time/

## 5. Follow-up work from Behrouz et al. (arXiv author search, https://arxiv.org/search/?query=Behrouz%2C+Ali&searchtype=author&order=-announced_date_first&size=50)

2026:
- 2602.24281 "Memory Caching: RNNs with Growing Memory", 27 Feb 2026, Behrouz, Zeman Li, Yuan Deng, Zhong, Razaviyayn, Mirrokni. Memory capacity that grows with sequence length. https://arxiv.org/abs/2602.24281
- 2606.03979 "Language Models Need Sleep: Learning to Self-Modify and Consolidate Memories", v1 2 Jun 2026, v2 10 Jul 2026, Behrouz, Hashemi, Javanmard, Mirrokni; comments: "A version of this work has been publicly available from September 2025 on OpenReview". Consolidation (distillation into a larger net) plus RL "Dreaming". https://arxiv.org/abs/2606.03979
- 2606.23670 "Tapered Language Models", 22 Jun 2026, Bayat, Behrouz, Courville; tapered MLP widths, evaluated on Titans among other backbones. https://arxiv.org/abs/2606.23670
- 2607.15271 "Online Neural Space Time Memory for Dynamic Novel View Synthesis", Jul 2026, Elmieh et al. incl. Behrouz. https://arxiv.org/abs/2607.15271
- 2608.16844 "Proteus: Incremental Memory Activation for Long-Context Sequence Modeling", 17 Aug 2026, Bayat, Behrouz, Mirrokni, Courville; applied to SWLA, Comba, Titans and Hope-Attention. https://arxiv.org/abs/2608.16844

Late 2025 (same group):
- 2505.23735 "ATLAS: Learning to Optimally Memorize the Context at Test Time", 29 May 2025; sizes 340M/400M/790M/1.3B; "While Atlas shows competitive and on par performance with Titans until 1M context length, the performance of Titans drops in 10M. Atlas, however, maintains its performance and achieve +80% accuracy in 10M context length." https://arxiv.org/abs/2505.23735
- 2511.07343 "TNT: Improving Chunkwise Training for Test-Time Memorization", 10 Nov 2025, Li, Behrouz, Deng, Zhong, Kacham, Karami, Razaviyayn, Mirrokni; 17x faster training of Titans-style memory. https://arxiv.org/abs/2511.07343
- 2512.23852 "Trellis: Learning to Compress Key-Value Memory in Attention Models" and 2512.23824 "MS-SSM", both 29 Dec 2025, Karami, Behrouz et al.

None of these arXiv pages announce code or weights.

## 6. Production status as of September 2026
- No primary source says Titans, MIRAS or Hope is in any shipped Google or open model. The two Google blog posts and the "Google Research 2025" wrap-up describe them as research advances only: https://research.google/blog/google-research-2025-bolder-breakthroughs-bigger-impact/
- Claims in secondary coverage that Google "plans to integrate Nested Learning into future Gemini models" could not be traced to any Google statement; the Verdict/Yahoo article cited for it only says Google "is seeking further exploration of this approach within the broader machine learning community": https://tech.yahoo.com/ai/gemini/articles/google-introduces-nested-learning-103210037.html
- What has shipped in open weights is the adjacent linear-RNN family, not Titans: Qwen3-Next-80B-A3B uses "the combination of Gated DeltaNet and Gated Attention" in a "12 * (3 * (Gated DeltaNet -> MoE) -> 1 * (Gated Attention -> MoE))" layout with 262,144 native context (https://huggingface.co/Qwen/Qwen3-Next-80B-A3B-Instruct ; config.json full_attention_interval 4). Qwen3.5 (0.8B to 397B-A17B) uses the same hybrid: Qwen3.5-9B config.json lists 24 linear_attention and 8 full_attention layers, max_position_embeddings 262144 (https://huggingface.co/Qwen/Qwen3.5-9B/raw/main/config.json). Gated DeltaNet is a delta-rule linear memory (the family MIRAS analyses as an L2 associative memory), not Titans' deep MLP memory with momentum and data-dependent decay, and neither model card mentions Titans.

## 7. Practical bottom line for a local Qwen on a Mac
There is nothing from this Google line you can install or download: no Titans, MIRAS or Hope weights, no official code, and the only Titans-on-Qwen artifact (TPTT) is a single-author LoRA demo evaluated on 500 Alpaca samples with no long-context test. If you already run Qwen3.5 or Qwen3-Next locally you already have the closest production-grade relative (a Gated DeltaNet plus attention hybrid with 262K native context); treat Titans and Hope as from-scratch research architectures to watch, and Aedelon/titans-pytorch-mlx or obekt/HOPE-nested-learning as toy-scale ways to experiment on Apple Silicon.

## 8. Method notes and limits of this check
- arXiv API and dblp were rate-limited or bot-gated during the check; the author list was taken from the arXiv search UI instead. Semantic Scholar's author search returned wrong people and was discarded.
- All plot values are visual readings of the figure PNGs from the arXiv HTML builds, accurate to a few percent.
- Hugging Face download counts were not available through the API call used, so repo popularity for TPTT checkpoints is not reported.
- Titans and Nested Learning have only v1 on arXiv; the Titans paper's full text is the Dec 2024 version, so any camera-ready NeurIPS changes are not reflected.
