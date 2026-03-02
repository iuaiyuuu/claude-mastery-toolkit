# Model Selection Decision Matrix

## By Task Type and Data Characteristics

### Tabular Data — Classification and Regression

| Scenario | First Choice | Second Choice | Avoid |
|---|---|---|---|
| General tabular, <100K rows | LightGBM/XGBoost | Random Forest, Logistic/Linear Regression | Deep learning |
| General tabular, >100K rows | LightGBM/XGBoost | CatBoost (if many categoricals) | Overly complex ensembles |
| High cardinality categoricals | CatBoost | LightGBM with target encoding | One-hot everything |
| Explainability required | Logistic/Linear Regression, EBMs | SHAP on gradient boosted trees | Uninterpretable ensembles without explanation layer |
| Very small data (<500 rows) | Logistic/Linear Regression, Bayesian methods | Regularized trees (shallow, few leaves) | Any deep learning |
| Streaming / online learning | Vowpal Wabbit, River, SGD-based models | Periodic retrain of batch models | Models requiring full dataset in memory |
| Very high dimensional (>10K features) | L1-regularized linear models, then boost | Feature selection → XGBoost | Unregularized models |

### NLP Tasks

| Scenario | First Choice | Second Choice | Avoid |
|---|---|---|---|
| Text classification, >5K labeled examples | Fine-tuned BERT/RoBERTa (base) | TF-IDF + LightGBM | Training transformers from scratch |
| Text classification, <500 examples | Few-shot with LLM API, SetFit | Fine-tuned small transformer with augmentation | Fine-tuning large models |
| Named entity recognition | Fine-tuned transformer NER (spaCy, HuggingFace) | CRF with handcrafted features | Regex-only approaches for complex entities |
| Semantic similarity | Sentence transformers (e.g., all-MiniLM) | Fine-tuned bi-encoder | TF-IDF cosine similarity (unless very simple) |
| Text generation / summarization | LLM API (GPT-4, Claude) for quality; fine-tuned T5/BART for cost | Smaller distilled models if latency-critical | Training generation models from scratch on small data |
| Multilingual | mBERT, XLM-R | Language-specific models if single-language | English-only models on multilingual data |

### Computer Vision

| Scenario | First Choice | Second Choice | Avoid |
|---|---|---|---|
| Image classification, standard | EfficientNet / ResNet with transfer learning | Vision Transformer (ViT) if >50K images | Training from scratch |
| Object detection | YOLOv8/v9 (real-time), DETR (accuracy) | Faster R-CNN | Custom architectures without strong justification |
| Semantic segmentation | U-Net variants, DeepLab | Segment Anything (SAM) for zero-shot | Complex architectures for simple segmentation tasks |
| Small dataset (<1K images) | Transfer learning with frozen backbone + fine-tuned head | Few-shot learning (prototypical networks, CLIP) | Full fine-tuning of large models |
| Edge deployment | MobileNet, EfficientNet-Lite | Quantized standard models | Models >100M parameters without distillation |

### Time-Series

| Scenario | First Choice | Second Choice | Avoid |
|---|---|---|---|
| Single univariate series | ARIMA/ETS (statistical) | Prophet (seasonality focus) | Deep learning |
| Many related series | LightGBM with lag features (global model) | N-BEATS, Temporal Fusion Transformer | Individual ARIMA per series |
| Long-horizon forecasting | Temporal Fusion Transformer, PatchTST | Hierarchical reconciliation + simpler models | Single-step models rolled forward |
| Irregular time-series | Neural ODEs, interpolation + standard models | GRU-D | Standard RNNs that assume regular spacing |
| Real-time / streaming | ARIMA with online updates, exponential smoothing | Lightweight neural models with incremental training | Batch-only models |

### Recommendation Systems

| Scenario | First Choice | Second Choice | Avoid |
|---|---|---|---|
| Collaborative filtering, dense interactions | Matrix factorization (ALS) | Neural collaborative filtering | Memory-based CF at scale |
| Sparse interactions | Two-tower models with content features | LightFM (hybrid) | Pure collaborative filtering |
| Sequential recommendations | Transformer-based (SASRec, BERT4Rec) | GRU4Rec | Non-sequential models when order matters |
| Cold-start dominant | Content-based with user/item features | Bandit approaches for exploration | Pure CF without fallback |
| Multi-objective (relevance + diversity) | Re-ranking layer on top of retrieval model | MMR (Maximal Marginal Relevance) | Single-objective optimization |

### Anomaly Detection

| Scenario | First Choice | Second Choice | Avoid |
|---|---|---|---|
| Tabular, unsupervised | Isolation Forest | Local Outlier Factor (LOF), One-Class SVM | Autoencoders on small tabular data |
| Time-series anomalies | Statistical process control, STL decomposition | LSTM-based reconstruction error | Models that don't account for temporal structure |
| High-dimensional | Autoencoders, VAE | PCA-based residuals | Distance-based methods in high dimensions |
| With some labeled anomalies | Semi-supervised: train on normal, validate with anomalies | XGBoost with severe class weighting | Ignoring the labels entirely |
| Network/graph anomalies | Graph neural network-based | Community detection + statistical outliers | Node-level methods ignoring graph structure |

## Model Complexity Ladder

When in doubt, follow this progression. Move to the next level only if the current one demonstrably fails:

1. **Rule-based / heuristic baseline** — always start here
2. **Simple statistical model** — linear/logistic regression, Naive Bayes, KNN
3. **Gradient boosted trees** — XGBoost, LightGBM, CatBoost
4. **Shallow neural networks** — MLPs, shallow CNNs, simple RNNs
5. **Deep pretrained models** — fine-tuned transformers, deep CNNs
6. **Large foundation models** — LLMs, large vision models, multi-modal
7. **Custom architectures** — only when domain-specific structure is well-justified

Each step up increases: compute cost, data requirements, debugging difficulty, and deployment complexity. Only move up when the performance gain justifies these costs.

## Ensemble Strategies

Ensembles are appropriate when:
- Single model performance is close to target but not quite there
- Different model types capture different patterns (e.g., tree + linear + neural)
- Prediction stability matters more than speed

Common patterns:
- **Stacking**: Train diverse base models, use a meta-learner to combine. Use cross-validated out-of-fold predictions to train the meta-learner.
- **Blending**: Simple weighted average of predictions. Tune weights on validation set.
- **Cascade**: Use a fast cheap model first; escalate uncertain predictions to a more expensive model.

Avoid ensembles when:
- Latency is critical and you can't afford multiple model calls
- The improvement is marginal (<1% relative) — maintenance cost isn't worth it
- Explainability is a hard requirement
