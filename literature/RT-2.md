# RT-2: Vision-Language-Action Models Transfer Web Knowledge to Robotic Control
By Brohan et al, DeepMind

Fine tune a pretrained VLM (e.g. PaLI-X/E) :
- Define token for robot actions 
- Co-Fine tune (robotic and classic VL task at the same time)
- Use RT-1 dataset for robotic examples
- 