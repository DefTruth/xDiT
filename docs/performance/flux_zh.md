## Flux.1性能表现

Flux.1是由Black Forest Labs推出的DiTs模型，它由Stable Diffusion的原班人马开发，包含三种变体：FLUX.1 [pro]、FLUX.1 [dev]和FLUX.1 [schnell]，均拥有12B参数。

Flux.1实时部署有如下挑战：

1. 高延迟：在单个A100上schnell 4 step采样生成2048px图片也需要10秒钟！这对dev和pro版本需要30~50 steps的情况延迟更加惊人。

2. VAE OOM：生成超过2048px的图片，在80GB VRAM的A100上VAE部分会出现OOM，即使DiTs主干有生成更高分辨图片分辨率能力，但是VAE已经不能承受图片之大了。

xDiT使用xDiT的混合序列并行USP+VAE Parallel来将Flux.1推理扩展到多卡。

xDiT还不支持Flux.1使用PipeFusion，因为schnell版本采样步数太少了，因为PipeFusion需要warmup所以不适合使用。
但是对于Pro和Dev版本还是有必要加入PipeFusion的，还在Work In Progress。

另外，因为Flux.1没用CFG，所以没法使用cfg parallel。



### 扩展性展示

在8xA100 (80GB) NVLink互联的机器上，USP最佳策略是把所有并行度都给Ulysses，生成1024px图片仅需0.93秒！生成2048px图片仅需2.63秒，相比单卡加速4.2倍。我们对更高分辨率图片生成加速效果更明显，比如4096px达到7.4x。


<div align="center">
    <img src="../../assets/performance/flux/flux_a100.jpg" 
    alt="latency-flux_a100">
</div>

在PCIe Gen4互联的8xL40上，4卡规模xDiT也有很好的加速。生成1024px图片，使用ulysses_degree=2, ring_degree=2延迟低于单独使用ulysses或者ring，1.21秒可以生图。
8卡相比反而会变慢，这是因为这是需要经过QPI通信。生成2048px，八卡也有4倍加速。我们预期使用PipeFusion会提升8卡的扩展性。

<div align="center">
    <img src="../../assets/performance/flux/flux_l40.jpg" 
    alt="latency-flux_l40">
</div>


### VAE Parallel

在A100上，单卡使用Flux.1超过2048px就会OOM。这是因为Activation内存需求增加，同时卷积算子引发memory spike，二者共同导致的。

使用通过Parallel VAE让xDiT得以一窥更高分辨率生成能力的真容，我们可以生成更高分辨率的图片。使用`--use_parallel_vae` 在[运行脚本中](../../examples/run.sh).

prompt是"A hyperrealistic portrait of a weathered sailor in his 60s, with deep-set blue eyes, a salt-and-pepper beard, and sun-weathered skin. He’s wearing a faded blue captain’s hat and a thick wool sweater. The background shows a misty harbor at dawn, with fishing boats barely visible in the distance."

2048px，3072px和4096px生成质量如下，可以看到4096px生成已经质量很低了。

<div align="center">
    <img src="../../assets/performance/flux/flux_image.png" 
    alt="latency-flux_l40">
</div>