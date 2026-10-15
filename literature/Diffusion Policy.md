# Diffusion Policy: Visuomotor Policy Learning via Action Diffusion
By Cheng Chi et al, 2024

## Motivations :
Change paradigm for robotic policy. THe tasks : learn the mapping observation -> action.
Challenge : Action conditional distribution can be multimodal + require high precision.
Previous attemps : explicit policy by action representation (e.g., gaussian mixture of categorical data -> regression) or implicit (a = arg min Energy function)

Approach : Diffusion models on the action space. Pros :
- Multimodality
- High-dimension -> allow prediction of sequence of actions 
- stability
Contribution :
- Closed-loop action sequences, "long" horizon control, and replanning options
- visual conditionnning
- Time-serie diffusion transformer, avoiding over-smoothing effects

## Design

Diffusion paradigm : DDPM/DDIM
Observation as conditionning : observe T_o time step in the past
Predict T_p actions (= K NFE of diffusion model) in the futur (simultaneously)
Re-plan after T_a actions

Diffusion model : 
- either 1D temporal CNN, condition on obs features following FiLM. Issue : time-smoothness inductive bias, so bad for high frequency actions
- or time-series transformer (based on MiniGPT) : best perfo but harder to train (in 2022)
Visual encoder : ResNet18

## Effect

- Multimodal behavior 
![[Pasted image 20261007165804.png]]

- Position control works better than velocity control, unlike previous work. This may be bc  position control is better for accurate control but encounter more multi modality, which is well handle by diffusionPolicy
- Nice scaling to high dimension allow sequence action prediction : helps temporal consistency
- Better training stability than energy-based models (Score mathcing vs InfoNCE loss)
- Connexion with control theory
