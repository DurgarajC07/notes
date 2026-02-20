Complete Interview Topics List: AI / ML / NLP / Data Science
Basic → Advanced | Intermediate → Senior Roles
1. AI System Design (Critical for Seniors)
RAG System Design
End-to-end RAG architecture
Vector DB selection (Pinecone, Weaviate, Milvus, Qdrant, ChromaDB, FAISS)
Indexing strategies: HNSW vs IVF vs Flat vs ScaNN
Hybrid search (dense + sparse retrieval)
BM25 + semantic search fusion (Reciprocal Rank Fusion)
Document chunking strategies (fixed-size, recursive, semantic, agentic)
Chunk overlap tuning
Metadata filtering and pre-filtering
Parent-child document retrieval
Multi-hop retrieval
Re-ranking models (Cross-Encoders, Cohere Rerank, ColBERT)
Query transformation (HyDE, step-back prompting, multi-query)
Handling 10M+ documents at scale
Embedding model selection and fine-tuning
Embedding dimensionality tradeoffs
Index sharding and partitioning
Retrieval evaluation (MRR, NDCG, Recall@K, Hit Rate)
RAG failure modes and debugging
Contextual compression
Multi-modal RAG (text + images + tables)
Graph RAG (knowledge graph + vector retrieval)
Corrective RAG, Self-RAG, Adaptive RAG
RAG vs Long Context Windows tradeoff
Citation and attribution in RAG
Caching strategies for repeated queries
Document freshness and incremental indexing
Real-Time Voice / Conversational Agents
End-to-end voice pipeline architecture
ASR (Automatic Speech Recognition) systems: Whisper, Deepgram, AssemblyAI
TTS (Text-to-Speech) systems: ElevenLabs, PlayHT, XTTS, Bark
VAD (Voice Activity Detection) tuning
Turn-taking logic and interruption handling
Barge-in detection
Endpointing strategies
TTFT (Time to First Token) optimization
Streaming ASR vs batch ASR
Streaming TTS and audio chunking
WebSocket vs WebRTC for real-time audio
Latency budgeting across the pipeline (ASR → LLM → TTS)
Silence detection and timeout handling
Echo cancellation and noise suppression
Emotion detection from voice
Speaker diarization
Multi-language and code-switching handling
Telephony integration (SIP, Twilio, Vonage)
Jitter buffer management
Audio codec selection (Opus, PCM, μ-law)
Conversation state management
Dialog management (slot filling, intent tracking)
Fallback and escalation strategies
High-Concurrency & Scalable Inference
GPU utilization and multi-GPU inference
vLLM and PagedAttention
Continuous batching vs static batching
KV cache management and memory optimization
Tensor parallelism vs pipeline parallelism vs data parallelism
Load balancing strategies for LLM inference
Autoscaling GPU instances
Request queuing and priority scheduling
Rate limiting and throttling
Model serving frameworks: TensorRT-LLM, Triton Inference Server, TGI, Ray Serve
Speculative decoding
Multi-model serving on same GPU
Cold start optimization
Serverless inference tradeoffs
Edge deployment vs cloud inference
Inference cost optimization
Token throughput vs latency tradeoffs
Prefill vs decode phase optimization
Data Quality & Leakage Prevention
Train/validation/test split strategies
Temporal data leakage (future information leakage)
Target leakage
Group-based splitting (prevent same entity in train and test)
Cross-validation strategies (K-Fold, Stratified, Time-Series, Group)
Data contamination in LLM benchmarks
Feature leakage detection
Data versioning (DVC, LakeFS)
Data lineage tracking
Reproducibility in experiments
System Design Patterns
Online vs offline inference
Batch vs real-time prediction serving
Feature store design (Feast, Tecton)
Event-driven ML architectures
Lambda vs Kappa architecture for ML
A/B testing infrastructure for ML
Multi-armed bandit for model selection
Feedback loop design
Human-in-the-loop systems
Model registry and governance
Multi-tenant ML platform design
ML Platform architecture (end-to-end)
Cost estimation and optimization for ML systems
2. NLP & Generative AI
Transformer Architecture (Deep Dive)
Self-attention mechanism (Q, K, V matrices)
Multi-head attention (why multiple heads)
Positional encoding (sinusoidal, learned, RoPE, ALiBi)
Layer normalization (Pre-norm vs Post-norm)
Feed-forward network role
Residual connections
Encoder-only vs Decoder-only vs Encoder-Decoder
Causal masking vs bidirectional attention
Attention complexity O(n²) and solutions
Flash Attention and memory-efficient attention
Grouped Query Attention (GQA) and Multi-Query Attention (MQA)
Mixture of Experts (MoE) architecture
Sliding window attention
Sparse attention patterns
KV cache mechanism
Scaling laws (Chinchilla, Kaplan)
Tokenization
Byte Pair Encoding (BPE)
WordPiece
Unigram (SentencePiece)
Byte-level BPE
Tokenizer training and vocabulary size tradeoffs
Token-to-cost relationship in API pricing
Handling multilingual tokenization
Out-of-vocabulary (OOV) handling
Sub-word tokenization advantages
Tokenization impact on model performance
Special tokens (BOS, EOS, PAD, CLS, SEP, MASK)
Chat templates and tokenizer formatting
LLM Fundamentals
Pre-training objectives (Causal LM, Masked LM, Seq2Seq)
Pre-training data curation and deduplication
Instruction tuning
RLHF (Reinforcement Learning from Human Feedback)
DPO (Direct Preference Optimization)
PPO vs DPO vs ORPO vs KTO
Constitutional AI and RLAIF
Emergent abilities in LLMs
In-context learning (ICL)
Chain-of-thought reasoning
Few-shot vs zero-shot vs one-shot prompting
Temperature, Top-K, Top-P (Nucleus sampling)
Beam search vs greedy vs sampling
Repetition penalty and frequency penalty
Logit bias
Structured output (JSON mode, function calling, grammar-constrained decoding)
Seed and reproducibility in generation
Prompt Engineering (Advanced)
System prompts, user prompts, assistant prompts
Chain-of-Thought (CoT)
Tree-of-Thought (ToT)
ReAct (Reasoning + Acting)
Self-consistency prompting
Prompt chaining
Meta-prompting
Prompt injection attacks and defenses
Jailbreaking techniques and guardrails
Dynamic prompt construction
Prompt compression
Multi-turn conversation design
Role-playing and persona design
Prompt evaluation and optimization
Few-shot example selection strategies
Retrieval-augmented prompting
Fine-Tuning LLMs
Full fine-tuning vs parameter-efficient fine-tuning (PEFT)
LoRA (Low-Rank Adaptation)
QLoRA
Adapter layers
Prefix tuning
Prompt tuning (soft prompts)
Fine-tuning data preparation and formatting
Catastrophic forgetting and mitigation
Learning rate scheduling for fine-tuning
Fine-tuning vs RAG decision framework
Evaluation of fine-tuned models
Overfitting detection in fine-tuning
Continual pre-training
Domain adaptation
Multi-task fine-tuning
Merge strategies (model merging, TIES, DARE)
Training frameworks: Hugging Face Transformers, Axolotl, LLaMA-Factory, Unsloth
LLM Evaluation
Perplexity
BLEU, ROUGE, METEOR
BERTScore
Human evaluation frameworks
LLM-as-judge (G-Eval)
MMLU, HellaSwag, ARC, TruthfulQA benchmarks
Task-specific evaluation
Hallucination detection and measurement
Factual consistency evaluation (SummaC, FactScore)
Toxicity and bias evaluation
Red teaming
Evaluation of RAG systems
Latency and throughput benchmarking
Cost per query analysis
Ragas framework for RAG evaluation
Arena-style evaluation (Chatbot Arena, ELO ranking)
Agentic AI & Tool Use
ReAct agent architecture
Function calling and tool use
Multi-agent systems (AutoGen, CrewAI, LangGraph)
Agent planning and decomposition
Agent memory (short-term, long-term, episodic)
Agent evaluation
Agent safety and guardrails
Orchestration frameworks (LangChain, LlamaIndex, Semantic Kernel)
State machines for agent workflows
Human-in-the-loop agent design
Agent error recovery and retry logic
Code generation agents
Browser/computer use agents
MCP (Model Context Protocol)
Classical NLP (Still Asked)
Text preprocessing (lowercasing, stemming, lemmatization, stop words)
TF-IDF
Word2Vec, GloVe, FastText
Named Entity Recognition (NER)
Part-of-Speech (POS) tagging
Dependency parsing
Sentiment analysis
Text classification approaches
Topic modeling (LDA, NMF)
Sequence labeling
Regex and rule-based NLP
Language detection
Spell correction and normalization
Coreference resolution
Relation extraction
Text summarization (extractive vs abstractive)
Question answering (extractive vs generative)
Machine translation fundamentals
N-gram language models
Embeddings (Deep Dive)
Word embeddings vs sentence embeddings vs document embeddings
Static vs contextual embeddings
Sentence-BERT (SBERT)
Contrastive learning for embeddings (SimCLR, CLIP)
Embedding fine-tuning for domain-specific tasks
Matryoshka embeddings
Dimensionality reduction of embeddings (PCA, t-SNE, UMAP)
Cross-encoder vs bi-encoder
Embedding similarity metrics (cosine, dot product, euclidean)
Multi-modal embeddings
Late interaction models (ColBERT)
Embedding quantization
Instruction-tuned embeddings (E5, BGE)
Pre-Transformer Models (Historical Context)
RNN, LSTM, GRU architectures
Seq2Seq with attention (Bahdanau, Luong)
Bidirectional RNNs
Encoder-decoder architecture evolution
CNN for text classification (TextCNN)
ELMo (contextualized embeddings before BERT)
BERT, GPT-1/2/3, T5, BART
Model family tree (GPT → ChatGPT → GPT-4, LLaMA, Mistral, etc.)
3. Core Machine Learning Theory
Fundamentals
Supervised vs unsupervised vs semi-supervised vs self-supervised learning
Reinforcement learning basics
Bias-variance tradeoff
Overfitting vs underfitting
Training, validation, test set philosophy
No free lunch theorem
Occam's razor in ML
Curse of dimensionality
Inductive bias
Parametric vs non-parametric models
Generative vs discriminative models
PAC learning (basic concept)
VC dimension (basic concept)
Regression
Linear regression
Polynomial regression
Ridge regression (L2)
Lasso regression (L1)
Elastic Net
Assumptions of linear regression
Multicollinearity (VIF)
Heteroscedasticity
Residual analysis
Classification
Logistic regression (sigmoid, decision boundary)
Softmax regression (multi-class)
Support Vector Machines (SVM) — linear and kernel
K-Nearest Neighbors (KNN)
Naive Bayes (Gaussian, Multinomial, Bernoulli)
Decision Trees (ID3, C4.5, CART)
Random Forests
Gradient Boosted Trees (XGBoost, LightGBM, CatBoost)
AdaBoost
Multi-class vs multi-label classification
One-vs-Rest, One-vs-One strategies
Calibration of classifiers (Platt scaling, isotonic regression)
Clustering
K-Means, K-Means++, Mini-batch K-Means
DBSCAN
Hierarchical clustering (agglomerative, divisive)
Gaussian Mixture Models (GMM)
Spectral clustering
Silhouette score
Elbow method
Cluster evaluation metrics (ARI, NMI, Davies-Bouldin)
Dimensionality Reduction
PCA (Principal Component Analysis)
t-SNE
UMAP
LDA (Linear Discriminant Analysis)
Autoencoders for dimensionality reduction
Feature selection vs feature extraction
Explained variance ratio
Ensemble Methods
Bagging vs boosting vs stacking
Random Forest internals
Gradient boosting internals
XGBoost, LightGBM, CatBoost differences
Hyperparameter tuning for tree-based models
Feature importance (Gini, permutation, SHAP)
Optimization & Training
Gradient descent (batch, mini-batch, stochastic)
Learning rate and learning rate schedules (step, cosine, warm-up)
SGD with momentum
Adam, AdamW, RMSProp, Adagrad
Gradient clipping
Weight initialization (Xavier, He, Kaiming)
Batch normalization
Layer normalization
Dropout
Early stopping
Vanishing and exploding gradient problems
Second-order optimization (Newton's method, L-BFGS — conceptual)
Mixed precision training (FP16, BF16)
Gradient accumulation
Gradient checkpointing
DeepSpeed, FSDP for distributed training
Loss Functions
MSE, MAE, Huber loss
Cross-entropy (binary, categorical)
Focal loss
Hinge loss
Contrastive loss, Triplet loss
KL divergence
Label smoothing
Custom loss functions
Regularization
L1 regularization (Lasso)
L2 regularization (Ridge)
Elastic Net
Dropout
Data augmentation
Early stopping
Weight decay
Noise injection
Mixup, CutMix
Spectral normalization
Evaluation Metrics
Accuracy, precision, recall, F1-score
Confusion matrix
ROC curve and AUC-ROC
Precision-Recall curve and AUC-PR
Log loss (cross-entropy loss)
Cohen's Kappa
Matthews Correlation Coefficient (MCC)
RMSE, MAE, MAPE, R² (for regression)
Lift and gain charts
Calibration curves
Statistical significance testing (t-test, bootstrap)
Metric selection for imbalanced datasets
Online vs offline evaluation
Interannotator agreement
Probability & Statistics (Foundations)
Bayes' theorem
Probability distributions (Normal, Bernoulli, Binomial, Poisson, Exponential, Uniform)
Central limit theorem
Maximum Likelihood Estimation (MLE)
Maximum A Posteriori (MAP)
Hypothesis testing (p-value, confidence intervals)
Type I and Type II errors
A/B testing and statistical significance
Bayesian vs Frequentist approach
Markov chains
Monte Carlo methods
Expectation-Maximization (EM) algorithm
Information theory (entropy, mutual information, KL divergence)
Correlation vs causation
Simpson's paradox
Sampling methods (stratified, reservoir, importance sampling)
Feature Engineering
Feature scaling (standardization, normalization, min-max)
Encoding categorical variables (one-hot, label, target, ordinal)
Feature crossing / interaction features
Polynomial features
Binning / discretization
Log transformation, Box-Cox, Yeo-Johnson
Handling missing values (imputation strategies)
Handling outliers
Feature selection methods (filter, wrapper, embedded)
Mutual information
Chi-squared test for feature selection
Recursive Feature Elimination (RFE)
Time-based features (lag, rolling, seasonal)
Text feature engineering (TF-IDF, n-grams, embedding features)
Automated feature engineering (Featuretools)
Handling Imbalanced Data
Oversampling (SMOTE, ADASYN)
Undersampling (random, Tomek links, NearMiss)
Class weights
Focal loss
Threshold tuning
Cost-sensitive learning
Ensemble methods for imbalanced data
Anomaly detection as alternative framing
4. Deep Learning
Neural Network Fundamentals
Perceptron and multi-layer perceptron (MLP)
Activation functions (ReLU, Leaky ReLU, GELU, Swish, Sigmoid, Tanh, Softmax)
Universal approximation theorem
Forward propagation
Backpropagation (chain rule, computational graph)
Weight initialization strategies
Batch size impact on training
Epochs vs iterations
Training dynamics and loss landscapes
Convolutional Neural Networks (CNN)
Convolution operation (filters, stride, padding)
Pooling (max, average, global average)
Feature maps and receptive field
Classic architectures (LeNet, AlexNet, VGG, ResNet, Inception, EfficientNet)
Skip connections / residual connections
1x1 convolutions
Transfer learning with pre-trained CNNs
CNN for text (TextCNN)
Recurrent Neural Networks
Vanilla RNN limitations
LSTM (gates: forget, input, output)
GRU
Bidirectional RNNs
Sequence-to-sequence models
Attention mechanism (Bahdanau, Luong)
Teacher forcing
Generative Models
Variational Autoencoders (VAE)
GANs (Generator, Discriminator, training dynamics)
Diffusion models (DDPM, Stable Diffusion basics)
Flow-based models (conceptual)
Mode collapse in GANs
FID, IS scores for generative model evaluation
Advanced Deep Learning
Attention mechanisms (self-attention, cross-attention)
Graph Neural Networks (GCN, GAT, GraphSAGE — conceptual)
Contrastive learning (SimCLR, CLIP)
Knowledge distillation
Neural Architecture Search (NAS — conceptual)
Multi-task learning
Transfer learning strategies
Meta-learning (few-shot learning)
Curriculum learning
Adversarial examples and robustness
5. MLOps & Production ML
Model Deployment
Model serialization (ONNX, TorchScript, SavedModel, GGUF, SafeTensors)
Containerization (Docker for ML)
Kubernetes for ML workloads
Model serving (TensorFlow Serving, TorchServe, Triton, BentoML, Ray Serve)
REST API vs gRPC for inference
Batch inference vs real-time inference
Serverless inference (AWS Lambda, SageMaker Serverless)
Edge deployment (TFLite, Core ML, ONNX Runtime Mobile)
Model compilation (TensorRT, Apache TVM, torch.compile)
Inference Optimization
Quantization (FP32 → FP16 → INT8 → INT4)
Post-training quantization (PTQ) vs quantization-aware training (QAT)
GPTQ, AWQ, GGUF quantization for LLMs
Model pruning (structured vs unstructured)
Knowledge distillation
Operator fusion
Dynamic batching
Caching strategies (prompt caching, semantic caching)
Speculative decoding
Flash Attention
Prefix caching
Model Monitoring & Observability
Model drift detection (data drift, concept drift, prediction drift)
Feature drift monitoring
Performance degradation alerting
Logging predictions and ground truth
A/B testing for models
Shadow deployment / dark launching
Canary releases for models
Model performance dashboards
Feedback loop implementation
Tools: Evidently AI, Whylabs, Arize, MLflow
ML Pipeline & Orchestration
ML pipeline design (training, evaluation, deployment)
Pipeline orchestration (Airflow, Kubeflow, Prefect, Dagster, ZenML)
Feature stores (Feast, Tecton, Hopsworks)
Data versioning (DVC, LakeFS)
Model versioning and registry (MLflow, Weights & Biases, Neptune)
Experiment tracking
Hyperparameter tuning (Grid, Random, Bayesian — Optuna, Ray Tune)
CI/CD for ML (GitHub Actions, Jenkins for ML)
Automated retraining triggers
Data validation (Great Expectations, Pandera)
ML metadata management
Infrastructure & Compute
GPU vs CPU vs TPU for training and inference
GPU memory management (OOM debugging)
Distributed training (data parallelism, model parallelism, pipeline parallelism)
DeepSpeed ZeRO stages
FSDP (Fully Sharded Data Parallel)
Mixed precision training
Spot instances and preemptible VMs for training
Cloud ML services (AWS SageMaker, GCP Vertex AI, Azure ML)
Cost optimization strategies
CI/CD for ML
Testing ML code (unit tests, integration tests, model tests)
Data quality tests
Model validation gates
Shadow deployment
Canary releases
Blue-green deployment
Rollback strategies
Automated model evaluation before deployment
Infrastructure as Code for ML
6. Data Engineering & Processing for ML
Data Pipelines
ETL vs ELT
Batch processing (Spark, Hadoop)
Stream processing (Kafka, Flink, Spark Streaming)
Data lake vs data warehouse vs data lakehouse
Apache Kafka fundamentals (topics, partitions, consumer groups, offsets)
Kafka for ML event streaming
Message ordering and exactly-once semantics
Data ingestion patterns
Schema evolution and management
Data Processing at Scale
Apache Spark (RDDs, DataFrames, Spark ML)
Pandas vs Polars vs Dask vs Modin
SQL for data science (window functions, CTEs, joins)
Data partitioning strategies
Handling large datasets that don't fit in memory
Parquet, Avro, ORC file formats
Delta Lake / Iceberg / Hudi
Data Quality
Data validation frameworks
Handling missing data at scale
Deduplication strategies
Data profiling
Outlier detection
Data labeling strategies and tools (Label Studio, Prodigy)
Active learning for efficient labeling
Weak supervision (Snorkel)
Synthetic data generation
Data augmentation techniques (text, image, tabular)
Data annotation quality assurance (IAA metrics)
Vector Databases (Deep Dive)
Vector DB architectures
Approximate Nearest Neighbor (ANN) algorithms
HNSW (Hierarchical Navigable Small World)
IVF (Inverted File Index)
Product Quantization (PQ)
ScaNN
Recall vs latency tradeoffs
Index tuning (nprobe, efSearch, efConstruction)
Filtering with metadata
Multi-tenancy in vector DBs
Benchmarking vector DBs
Hybrid search implementation
7. Computer Vision (If Applicable)
Image classification
Object detection (YOLO, Faster R-CNN, DETR)
Image segmentation (semantic, instance, panoptic)
Image generation (GANs, Diffusion models, Stable Diffusion)
Vision Transformers (ViT)
CLIP and multi-modal models
OCR and document understanding
Video understanding
Data augmentation for images
Transfer learning with pre-trained vision models
Image embeddings
Few-shot image classification
Multi-modal LLMs (GPT-4V, LLaVA, Gemini Vision)
Document parsing (layout analysis, table extraction)
8. Recommender Systems
Content-based filtering
Collaborative filtering (user-based, item-based)
Matrix factorization (SVD, ALS)
Deep learning for recommendations (NCF, Wide & Deep, Two-Tower)
Session-based recommendations
Cold start problem
Implicit vs explicit feedback
Evaluation metrics (NDCG, MAP, Hit Rate, MRR)
Multi-objective ranking
Exploration vs exploitation
Real-time recommendation serving
Embedding-based retrieval for recommendations
LLM-powered recommendations
9. Time Series & Forecasting
Time series decomposition (trend, seasonality, residual)
Stationarity (ADF test, KPSS test)
ARIMA, SARIMA
Exponential smoothing (Holt-Winters)
Prophet
LSTM / GRU for time series
Temporal Fusion Transformer
Feature engineering for time series (lags, rolling stats, Fourier features)
Cross-validation for time series (walk-forward, expanding window)
Anomaly detection in time series
Multi-step forecasting strategies
Evaluation (MAPE, SMAPE, RMSE, MASE)
Causal inference in time series
10. Responsible AI, Ethics & Safety
Bias detection and mitigation in ML models
Fairness metrics (demographic parity, equalized odds)
Explainability / Interpretability (SHAP, LIME, attention visualization)
Model cards and documentation
AI safety and alignment
Guardrails for LLMs (content filtering, output validation)
PII detection and redaction
Differential privacy
Federated learning
Adversarial attacks and defenses
Prompt injection and defense strategies
Regulatory compliance (GDPR, EU AI Act, SOC 2)
Toxicity detection
Watermarking AI-generated content
Copyright and training data concerns
Red teaming LLMs
Hallucination grounding and factuality
11. Information Retrieval & Search
TF-IDF and BM25
Inverted index
Semantic search
Lexical vs semantic vs hybrid search
Learning to Rank (LTR)
Pointwise, pairwise, listwise ranking
Query understanding (intent classification, query expansion)
Relevance feedback
Search evaluation (NDCG, MRR, MAP, Precision@K, Recall@K)
Faceted search
Autocomplete and query suggestion
Spell correction in search
Personalized search
Entity linking
Knowledge graphs for search
12. Tabular Data & Classical Data Science
Exploratory Data Analysis (EDA)
Hypothesis testing for business problems
A/B testing design and analysis
Causal inference (propensity score matching, DiD, instrumental variables)
Survival analysis
Customer churn prediction
Customer Lifetime Value (CLV)
Fraud detection
Credit scoring
Uplift modeling
Market basket analysis (association rules, Apriori)
Anomaly / outlier detection (Isolation Forest, LOF, One-Class SVM)
13. Scenario-Based & Behavioral Questions
Technical Scenarios
Debugging model performance degradation in production
Handling LLM hallucinations in production
Fixing VAD interruption issues in voice agents
Scaling Kafka consumers during peak inference
Reducing inference latency for real-time applications
Handling cold start in recommendation/prediction systems
Debugging data pipeline failures
Migrating from one model/framework to another
Responding to model bias discovered post-deployment
Cost optimization when cloud bills spike
Leadership & Architecture Scenarios
Choosing build vs buy for ML components
Prioritizing ML projects with business impact
Mentoring junior ML engineers
Cross-functional collaboration (product, engineering, data)
Communicating ML concepts to non-technical stakeholders
Making tradeoff decisions (accuracy vs latency vs cost)
Post-mortem analysis for ML system failures
Planning ML roadmaps
Technical debt management in ML systems
STAR Method Topics
Most challenging ML project
A time you dealt with ambiguous requirements
How you improved model performance significantly
How you handled disagreements on technical approach
A production incident you resolved
How you balanced speed vs quality
14. Math Foundations (Commonly Tested)
Linear Algebra
Vectors, matrices, tensors
Matrix multiplication and its role in neural networks
Eigenvalues and eigenvectors
Singular Value Decomposition (SVD)
Matrix rank
Dot product and cosine similarity
Orthogonality
Calculus
Derivatives and partial derivatives
Chain rule (backpropagation foundation)
Gradients and Jacobians
Hessian matrix (conceptual)
Convex vs non-convex optimization
Probability
Conditional probability
Bayes' theorem applications
Common distributions
Expectation, variance, covariance
Law of large numbers
Central limit theorem
Markov property
15. Programming & Coding for ML
Python proficiency (data structures, OOP, decorators, generators)
NumPy operations and broadcasting
Pandas data manipulation
SQL (complex queries, window functions, optimization)
PyTorch fundamentals (tensors, autograd, DataLoader, training loops)
TensorFlow / Keras basics
Hugging Face Transformers library
LangChain / LlamaIndex
Scikit-learn pipeline
Coding ML algorithms from scratch (logistic regression, decision tree, k-means)
Coding attention mechanism from scratch
Data structure & algorithm questions adapted for ML
Writing efficient data processing code
Debugging ML training code
Unit testing ML code
API development for ML (FastAPI, Flask)
Async programming for ML serving
16. Domain-Specific Knowledge
Conversational AI / Dialog Systems
Dialog state tracking
Intent classification and entity extraction
Slot filling
Dialog policy learning
Conversation flow design
Multi-turn context management
Personality and tone consistency
Grounding and knowledge integration
Error handling and recovery in dialogs
Evaluation of dialog systems (task completion, user satisfaction)
Document AI / Form Processing
Document layout analysis
Table extraction
Key-value pair extraction
Document classification
Handwriting recognition
Invoice / receipt processing
PDF parsing strategies
Multi-page document understanding
Document question answering
Healthcare / Finance / Legal AI (Domain-Specific)
Compliance requirements per domain
Domain-specific evaluation metrics
Specialized pre-training and fine-tuning
Handling sensitive data
Explainability requirements
Regulatory frameworks
17. Emerging & Advanced Topics
Multi-modal models (text + image + audio + video)
Small Language Models (SLMs) and on-device AI
Retrieval-Augmented Generation advances
Long context models (1M+ tokens)
Reasoning models (o1, o3, DeepSeek-R1)
Test-time compute scaling
Compound AI systems
Agentic workflows and autonomous agents
Model merging and model soups
Synthetic data generation at scale
Reinforcement Learning from AI Feedback (RLAIF)
Constitutional AI
Instruction hierarchy
Distillation of large models to small models
Open-source vs closed-source LLM landscape
LLM routers and model selection
MCP (Model Context Protocol) and A2A
Structured generation (Outlines, Instructor, LMQL)
Evaluation-driven development for LLMs
AI coding assistants (Copilot, Cursor, Devin)
Neurosymbolic AI
World models
