# Inside Multimodal Foundation Models: Representations as a Lever for Control, Safety and Grounding
Mercredi 07 Octobre 2026

Insights of multimodals Foundation models -> Representation

Scale is not the only Lever : the failure of MLLMs (counting, spatial relations, dense scenes...) are not a data problem

Inside : layers of transformers. Challenges : reading, steering, 

READ, ENRICH, STEER, READ OUT

## I. Read, locate the structure

MLLMs = LLM + Visual Encoder

Challenge when training MLLMs : Align representation (image and texts)
Alignement regularize M

Work : Mind the Head, topological repr alignement 

Check (via KNN) that features of LLM and visual enc. are aligned -> do it at att. head level (atomic level)

Between dinov2 and Qwen2.5 : in each layer, huge spread on head, so head is the right unit, not the layer average

Then try to align heas with some loss function :  a contrastive loss based on knn (note: knn is not diff, so use a diff proxy for knn : L_RA)
Act as a regularizer for classic cross entropy loss in MLLM training

Ok the training is a bit better 

## II. Enrich, add and refine evidence

beyond caption : Knowledge based VQA (multi modal multi-hop)

2 RAG works :
Hierarchical Retrieval for RAG 
Reasoning over Retrieved Knowledge
=> Performance is based retrieved perf

Can we do better by building a latent space for retrieval
Universal multimodal Retrieval : the query AND the retrieval are multimodals:
2 encoders : Vision and texts : mix the representations at different layers and shares archi between query and retrieval

## III. Steer, move behaviour in latent space

Selective Safety Alignment : CLip underlies generation, retrieval, MLLMs, but wbe data can bring unsafe asssociations
Act in the emùbedding space : move toward "safe space" while not destroying too much the original embeddings 

VisuV2 datasets : quadruplets with human valided safety labels

Transfer safety : identify a "safety" vector and transfer it


## IV.Read out,  Extract the carrier subspace

Few channel draw the whole picture. Can extract activation from a NFE and merge it with other to interpolate 2 images, or image and text