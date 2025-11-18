---
marp: true
theme: default
paginate: true
---

# FineWeb: Decanting the Web for the Finest Text Data at Scale

**Authors:** Guilherme Penedo, Hynek Kydlíček, Loubna Ben Allal, Anton Lozhkov, Colin Raffel, Leandro Werra, Thomas Wolf

**Affiliation:** HuggingFace

**Published:** May 31, 2024

![bg right:40% 80%](screenshot.jpeg)

---

## What is FineWeb?

- **Large-scale dataset** for LLM pretraining
  - **15 trillion tokens** (44TB disk space)
  - Derived from **96 CommonCrawl snapshots**

- **Best-performing** open pretraining dataset
  - Outperforms RefinedWeb, C4, Dolma, The Pile, SlimPajama

- **Fully documented** and ablated design choices
  - Reproducible processing pipeline
  - Released under permissive **ODC-By 1.0 license**

---

## FineWeb-Edu: Educational Subset

![width:900px](screenshot.jpeg)

- **Filtered for educational quality** using LLM annotations
- Two versions available:
  - **1.3 trillion tokens** (very high educational content, score ≥ 3)
  - **5.4 trillion tokens** (high educational content, score ≥ 2)
- **10x more efficient** than C4 on educational benchmarks
- Outperforms all open datasets on MMLU, ARC, OpenBookQA

---

## Where Does the Data Come From?

**CommonCrawl** - Non-profit web crawling organization
- Crawling the web since 2007
- New crawl every 1-2 months (200-400 TiB each)
- Latest crawl (April 2024): **2.7 billion web pages**, **386 TiB** HTML

**Why CommonCrawl?**
- Public repository of crawled webpages
- Alternative: crawl yourself (like OpenAI, Anthropic)
- 96 crawls released since 2013

---

## Processing at Scale

**Challenge:** Process hundreds of terabytes efficiently

**Solution: DataTrove**
- Open-source data processing library
- Seamlessly scales to **thousands of CPU cores**
- Modular and easily iterable design
- All FineWeb processing scripts available

---

## What is "Good Data"?

**Problem:** "High quality" is not well-defined for LLM pretraining

**Evaluation Approach:**
1. Train **small models** (1.82B parameters) on dataset variants
2. Evaluate on diverse **benchmark tasks**
3. Compare average scores

**Ablation Setup:**
- Llama architecture, 2048 sequence length
- Trained on **~28B tokens** (Chinchilla optimal for 1.8B)
- Longer validation runs on **350B tokens**
- Evaluation using **lighteval** framework

---

## Evaluation Benchmarks

Selected for good signal at small scale:
- **CommonSense QA**
- **HellaSwag**
- **OpenBook QA**
- **PIQA**
- **SIQA (Social IQA)**
- **WinoGrande**
- **ARC**
- **MMLU**

**Criteria:**
- Small variance between runs
- Monotonic performance increase
- Above random baseline

---

## The FineWeb Recipe

### 1. Text Extraction

**From WARC files** (not WET)
- Used **trafilatura library** for extraction
- WET files use suboptimal default extraction
- WARC extraction produces **25% smaller but higher quality** dataset

**Trade-off:** Text extraction is most costly step
- WET acceptable for lower budget teams

---

## The FineWeb Recipe

### 2. Base Filtering

Following RefinedWeb approach:
- **URL filtering** using blocklist (remove adult content)
- **Language filtering** - fastText classifier (English ≥ 0.65 score)
- **Quality & repetition filters** from MassiveText

**Result:** ~36 trillion tokens after base filtering

---

## Deduplication: Why?

**The Web is Full of Duplicates:**
- Aggregators, mirrors, templated pages
- Repeated content across domains
- Crawler-introduced duplicates (same page, different links)

**Benefits of Deduplication:**
- **Improved model performance**
- **Reduced memorization** → better generalization
- **Training efficiency** - same performance with fewer iterations
- **More diverse data** for given number of training tokens

---

## Deduplication Approach

**Method: MinHash** (fuzzy hash-based deduplication)

**Parameters:**
- Collect 5-grams from each document
- 112 hash functions (14 buckets × 8 hashes)
- Target: **≥75% similarity threshold**

**Probability of matching as duplicates:**
- 70% similar: 56% match probability
- 75% similar: 77% match probability
- 80% similar: 92% match probability
- 85% similar: 98.8% match probability

---

## Deduplication: Surprising Discovery

**Initial Assumption:** More deduplication = better performance

**Experiment 1: Global deduplication across all dumps**
- Deduplicated all 90+ dumps together
- Removed 90%+ from oldest dumps
- Result: **4 trillion tokens**
- Performance: **NO improvement** over non-deduplicated data!

**Why?**
- Data kept in oldest dumps was actually **worse quality**
- Deduplication **upsampled lower quality data**

---

## Deduplication: The Solution

**New Approach: Independent deduplication per dump**

- Deduplicate each dump **individually**
- Result: **20 trillion tokens**
- Performance: **Matches RefinedWeb!**

**Hypothesis:**
- Main benefit: removing very large duplicate clusters (100K+ documents)
- Further deduplication of small clusters (<100 duplicates) **harms performance**
- Data unique to a dump may be out-of-distribution/lower quality

---

## Failed Global Deduplication Attempts

Tried additional deduplication on top of independent MinHash:

1. **URL deduplication** (5.6T left) - worse performance
2. **Line deduplication** (4.4T left) - worse performance
3. **Line dedup with min words** (2.9T left) - worse performance
4. **3-line span deduplication** (3.7T left) - worse performance

**Conclusion:** Independent MinHash per dump is optimal

---

## C4 Filters Analysis

**C4 Dataset** (2019) still shows strong performance

**Applied C4 filters individually:**
- **Terminal punctuation filter** - biggest boost, but removes 30% of tokens
- **Curly bracket filter** - small boost, removes 2.8%
- **Word lengths filter** - small boost, removes 4.3%
- JavaScript, lorem ipsum, policy filters - each <0.5% removed

**Decision:** Apply all C4 filters **except** terminal punctuation
- Better performance with less data removal (~7% vs 30%)

---

## Statistical Filter Development

**Systematic process:**
1. Collect **50+ high-level statistics** on datasets
2. Compare high-quality vs low-quality web data
3. Select metrics with largest **Wasserstein distance**
4. Choose thresholds empirically
5. Validate with ablation runs

**Comparison datasets:**
- Independent MinHash (high quality)
- Global MinHash (lower quality)

---

## Custom Filters

**Three winning filters:**

1. **Fraction of lines ending with punctuation ≤ 0.12**
   - Removes lists, layout text ("Home", "Sign up")
   - Removes: 10.14% of tokens

2. **Fraction of chars in duplicate lines ≥ 0.1**
   - Tighter than MassiveText threshold (0.2)
   - Removes: 12.47% of tokens

3. **Fraction of short lines (<30 chars) ≥ 0.67**
   - Removes: 3.73% of tokens

**Combined:** ~22% of tokens removed, significant performance boost

---

## Final FineWeb Pipeline

**Processing steps in order:**
1. Text extraction from WARC (trafilatura)
2. Base filtering (URL, language, quality)
3. Independent MinHash per dump
4. Selected C4 filters
5. Custom statistical filters

**Result:** **15 trillion tokens** of high-quality data

Each step provides measurable performance improvement!

---

## Performance Comparison

**Datasets compared:**
- RefinedWeb (500B tokens)
- C4 (172B tokens)
- Dolma v1.6 (3T tokens)
- The Pile (340B tokens)
- SlimPajama (627B tokens)
- RedPajama2 (20T tokens)
- **FineWeb (15T tokens)** ← Winner!

**FineWeb is the best-performing open dataset** for LLM pretraining

---

## FineWeb-Edu: Annotation Process

**Goal:** Filter for educational content at scale

**Method:**
1. Used **Llama-3-70B-Instruct** to annotate 500K samples
2. Educational score: **0 to 5** (additive scale)
3. Focused on grade-school and middle-school knowledge
4. Avoided overfitting to highly technical content

**Why Llama-3-70B?**
- Tested Mixtral-8x7B, Mixtral-8x22B, Llama-3-70B
- Llama-3 alone gave most reliable results

---

## FineWeb-Edu: Classifier Training

**Scaling to trillions of tokens:**

**Model:** Snowflake-arctic-embed + classification head
- Trained on 450K Llama-3 annotations
- 20 epochs, learning rate 3e-4
- Frozen embedding and encoder layers
- Binary classification with threshold = 3
- **F1 score: 82%** on validation set

**Inference:** 6,000 H100 GPU hours to process 15T tokens

**Classifier available:** HuggingFaceFW/fineweb-edu-classifier

---

## FineWeb-Edu: Results

**Threshold experiments:**
- Threshold > 3: Better on knowledge benchmarks, worse on HellaSwag/PIQA
- **Threshold = 3: Best overall balance**

**Filtering results:**
- Removed **92%** of dataset
- **1.3 trillion tokens** of educational content remaining

**Performance:**
- **Surpasses all open web datasets**
- **10x more efficient** - matches C4 MMLU with 10x fewer tokens
- Exceptional on MMLU, ARC, OpenBookQA

---

## Bonus: CommonCrawl Over Time

**Discovery:** Not all crawls are equal!

**Experiment:**
- Trained 192 models (1.8B each) on 27B tokens
- Two runs per crawl with different samplings
- 60,000+ H100 GPU hours total

**Findings:**
- **Significant variance** between dumps
- Recent crawls (2023-2024) perform better
- Could not identify conclusive explanation

---

## Synthetic Data in CommonCrawl

**Hypothesis:** Recent crawls contain more LLM-generated content

**Proxy metric - ChatGPT phrases:**
- "delve", "as a large language model"
- "it's important to note", "rich tapestry"
- "intertwined", "certainly!", "dive into"

**Results:**
- Frequency **constant until 2023-14**
- **Steep increase** after ChatGPT release (late 2022)
- Synthetic data doesn't seem to harm performance
- Might even improve it for smaller trainings

**Uncertainty:** Effect on much larger trainings unknown

---

## Key Takeaways

1. **Text extraction matters** - WARC + trafilatura > WET
2. **More deduplication ≠ better** - independent per dump is optimal
3. **Statistical filtering works** - data-driven threshold selection
4. **Educational filtering is powerful** - 10x efficiency gains
5. **Recent crawls perform better** - possibly due to synthetic data
6. **Evaluation at scale is critical** - small ablations can be misleading

---

## Impact & Availability

**FineWeb Dataset:**
- 15 trillion tokens, 44TB
- HuggingFaceFW/fineweb

**FineWeb-Edu:**
- 1.3T tokens (score ≥ 3): HuggingFaceFW/fineweb-edu
- 5.4T tokens (score ≥ 2): HuggingFaceFW/fineweb-edu-score-2

**Tools:**
- DataTrove processing library
- Educational classifier: HuggingFaceFW/fineweb-edu-classifier

**License:** ODC-By 1.0 (permissive)

---

## Future Directions

**Short term:**
- Apply learnings to **other languages**
- Multilingual high-quality web data

**Long term:**
- Continue iterating on FineWeb
- Release better filtered subsets
- Open and reproducible dataset science

**Vision:**
> "The future is bright and exciting for studying the science of creating datasets at scale and in the open"

---

## Citation

```bibtex
@inproceedings{
  penedo2024the,
  title={The FineWeb Datasets: Decanting the Web
         for the Finest Text Data at Scale},
  author={Guilherme Penedo and Hynek Kydlíček and
          Loubna Ben allal and Anton Lozhkov and
          Margaret Mitchell and Colin Raffel and
          Leandro Von Werra and Thomas Wolf},
  booktitle={The Thirty-eight Conference on Neural
             Information Processing Systems Datasets
             and Benchmarks Track},
  year={2024},
  url={https://openreview.net/forum?id=n6SCkn2QaG}
}
```

---

## Thank You!

**Resources:**
- Dataset: https://huggingface.co/datasets/HuggingFaceFW/fineweb
- Blog: https://huggingface.co/spaces/HuggingFaceFW/blogpost-fineweb-v1
- DataTrove: https://github.com/huggingface/datatrove

**Questions?**
