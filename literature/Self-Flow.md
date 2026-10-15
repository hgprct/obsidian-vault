# Self-Supervised Flow Matching for Scalable Multi-Modal Synthesis
Black Forest Labs

## Motivation
For image/video generation, having semantic internal representation yield faster and better generation (see REPA work). Should help for coarse scene generation (understanding the links betweeen objects)
Thus previous work tries to align representation with pretrained encoders (mainly Dino)
Issue : Scaling is hard, constrained by avialable encoders on the given modality, compatibility is hard to predict

Sefl-Flow tries to obtained semantic internal representation alone, no external model reference

## Dual-Timestep Scheduling (DTS)

![[Pasted image 20261008121456.png]]

Use two different noise level inside a sample data : 
![[Pasted image 20261008121600.png|394]]


Training obj : Classical Flow mathcing loss (rectified flow) + representation loss 
![[Pasted image 20261008121716.png|386]]


### What to choose for $p(t)$ ?
![[Pasted image 20261008121909.png]]

