# Data Requirements by Task Type

## Minimum Data Guidelines

These are practical minimums for reasonable model performance, not theoretical lower bounds. More data almost always helps, but below these thresholds, expect significant challenges.

### Binary Classification
- **Minimum**: 500-1,000 samples per class for traditional ML (logistic regression, gradient boosted trees)
- **Comfortable**: 5,000+ per class
- **With class imbalance**: At least 200 samples of the minority class; use stratified sampling, SMOTE with caution (only on training set), or class weights
- **Deep learning**: 10,000+ total samples minimum, more for complex inputs

### Multi-class Classification
- **Minimum**: 100-500 samples per class for gradient boosted trees with good features
- **Comfortable**: 1,000+ per class
- **Many classes (100+)**: Consider hierarchical classification or embedding-based approaches
- **Zero-shot or few-shot**: If some classes have <50 samples, use transfer learning or label embeddings

### Regression
- **Minimum**: 500-1,000 samples for linear models with well-engineered features
- **Comfortable**: 5,000+ for nonlinear relationships
- **High-dimensional features**: Need ~10x samples per feature dimension for traditional models
- **Deep learning regression**: 10,000+ minimum

### Time-Series Forecasting
- **Minimum**: 3-5 complete seasonal cycles (e.g., 3-5 years for annual seasonality)
- **Short series**: Use simple models (exponential smoothing, ARIMA) or global models trained across multiple series
- **Many short series**: Pool data across series for global models (LightGBM, N-BEATS, temporal fusion transformers)
- **Single long series**: At least 500 time steps for deep learning approaches

### NLP Tasks
- **Text classification**: 500-1,000 examples per class with fine-tuned transformers; 50-100 with few-shot prompting
- **Named entity recognition**: 1,000+ annotated sentences with representative entity coverage
- **Sentiment analysis**: 2,000+ labeled examples across sentiment categories
- **Summarization/generation**: 5,000+ input-output pairs for fine-tuning; consider using LLM APIs for smaller datasets

### Computer Vision
- **Image classification**: 500-1,000 images per class with transfer learning from ImageNet-pretrained models
- **Object detection**: 1,000+ annotated images with diverse viewpoints and scales per target class
- **Segmentation**: 500+ pixel-level annotated images (annotation is expensive — budget accordingly)
- **Few-shot**: As few as 5-20 examples per class with proper few-shot learning architectures or foundation models

### Recommendation Systems
- **Collaborative filtering**: At least 20 interactions per user/item on average; very sparse matrices need content features
- **Content-based**: Feature quality matters more than volume; 1,000+ items with good metadata
- **Cold-start**: Always plan for new users/items with content-based fallbacks

### Anomaly Detection
- **Unsupervised**: 1,000+ normal samples to learn the distribution; anomalies not required for training
- **Semi-supervised**: 10,000+ normal samples; a small set (50-100) of labeled anomalies helps with threshold tuning
- **Supervised (if available)**: Treat as imbalanced classification

## Handling Insufficient Data

When data falls below minimums, apply these strategies in order of preference:

### 1. Simplify the Model
Reduce model complexity to match data volume. Linear models, shallow trees, or Naive Bayes may outperform complex models on small datasets. Fewer parameters = less data needed.

### 2. Transfer Learning
Use pretrained models as feature extractors or starting points:
- **NLP**: Fine-tune a pretrained language model (BERT, RoBERTa, or use LLM embeddings)
- **CV**: Use ImageNet-pretrained CNNs (ResNet, EfficientNet) as backbones
- **Tabular**: Less established, but pretrained embeddings for categorical features (e.g., entity embeddings) can help

### 3. Data Augmentation
Create synthetic training samples:
- **Images**: Rotation, flipping, cropping, color jittering, mixup, cutout. Use augmentation libraries (albumentations, torchvision transforms). Don't augment the validation/test set.
- **Text**: Back-translation, synonym replacement, contextual augmentation via language models. Be careful with label-preserving augmentation.
- **Tabular**: SMOTE (only on training folds), Gaussian noise injection, feature-space interpolation. Less reliable than image/text augmentation.
- **Time-series**: Window slicing, magnitude warping, time warping. Ensure augmented samples preserve temporal structure.

### 4. Semi-supervised and Self-supervised Learning
Use unlabeled data to improve representations:
- Pretrain on unlabeled data (masked language modeling, contrastive learning), then fine-tune on labeled data
- Pseudo-labeling: Train on labeled data, predict on unlabeled data, retrain with high-confidence pseudo-labels. Use cautiously — error propagation is real.

### 5. Active Learning
When labeling budget is limited, label the most informative samples first:
- Uncertainty sampling: Label samples where the model is least confident
- Diversity sampling: Label samples that are most different from already-labeled data
- Expected model change: Label samples that would most change the model

### 6. Synthetic Data Generation
Use generative models or simulation to create training data:
- Effective for structured/tabular data when the generative process is well-understood
- Use domain simulators for robotics, autonomous driving, medical imaging
- Always validate that synthetic data distribution matches real-world distribution
- Never rely solely on synthetic data — always include real data in validation

## Data Quality vs. Quantity

Data quality frequently matters more than quantity. Common quality issues that degrade performance more than small dataset size:

- **Label noise**: 10% label noise can negate doubling dataset size. Invest in annotation quality (clear guidelines, multiple annotators, adjudication).
- **Selection bias**: Training on a non-representative subset is worse than less data from a representative sample.
- **Feature leakage**: A large dataset with leaked features produces misleadingly good results. Always audit.
- **Stale data**: Data from a different distribution (last year's patterns for today's predictions) can hurt more than help. Recency often beats volume.
