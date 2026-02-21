---
layout: post
title:  "Local LLMs on Titan RTX"
date:   2026-02-21 14:30:00
tags: ai ollama
---

After getting my hands on a Titan RTX with 24GB of VRAM, [shoutout to the un-imaginatively named [PC Server and Parts](https://pcserverandparts.com/)] I decided to move past hosted AI services and see what it could do in a Lenovo P920 with Rocky Linux.

<a data-flickr-embed="true" href="https://www.flickr.com/photos/wsmoak/55081818171/in/dateposted-public/" title="260206_2038"><img src="https://live.staticflickr.com/65535/55081818171_f411e46d19_z.jpg" width="640" height="313" alt="260206_2038"/></a><script async src="//embedr.flickr.com/assets/client-code.js" charset="utf-8"></script>

## The Stack

The goal was a clean, containerized setup that could leverage the GPU without cluttering the host OS:

OS: Rocky Linux 9

Engine: Ollama running in Podman

Frontend: Open WebUI

Hardware: NVIDIA Titan RTX (24GB VRAM)

## Basic Setup with Podman

I started by running Ollama as a container. Since I’m on Rocky, Podman is the natural choice.

```bash
podman run -d --name ollama \
  --device nvidia.com/gpu=all \
  -v ollama:/root/.ollama \
  -p 11434:11434 \
  ollama/ollama
```

Initially, I tried using the --network host flag for Open WebUI, but Rocky’s firewalld and rootless Podman networking didn't play nice with external access. I eventually landed on explicit port mapping to get the UI visible on the network:

```bash
podman run -d -p 9091:8080 \
  --add-host=host.containers.internal:host-gateway \
  -v open-webui:/app/backend/data \
  -e OLLAMA_BASE_URL=http://host.containers.internal:11434 \
  --name open-webui \
  ghcr.io/open-webui/open-webui:main
```

## The VRAM Reality Check

Gemini enthusiasically recommended Llama 3.3 70B for this setup.  The experience was... meditative. It was generating at about one word per second. Digging into the logs, I found the culprit:

```bash
$ podman logs ollama 2>&1 | grep -i "offload"
load_tensors: offloaded 29/81 layers to GPU
```

A 4-bit quantized 70B model needs about 42GB of VRAM to run entirely on the GPU. With my 24GB card, Ollama was offloading 65% of the work to my CPU and System RAM. The PCIe bus became the bottleneck.

<div style="padding:60.5% 0 0 0;position:relative;">
<iframe src="https://player.vimeo.com/video/1167003067?badge=0&amp;autopause=0&amp;player_id=0&amp;app_id=58479" frameborder="0" allow="autoplay; fullscreen; picture-in-picture; clipboard-write; encrypted-media; web-share" referrerpolicy="strict-origin-when-cross-origin" style="position:absolute;top:0;left:0;width:100%;height:100%;" title="lenovo-p920-titan-rtx-ollama-llama-33-70b-20260221"></iframe>
</div>
<script src="https://player.vimeo.com/api/player.js"></script>

## Finding the Sweet Spot

To get that "instant" AI feel, the entire model needs to fit in VRAM. I experimented with three tiers:

Llama 3.1 8B: Lightning fast (100+ tokens/sec). Great for simple tasks.

Gemma 2 27B: It fits 100% in VRAM with room for a long context window. It's significantly smarter than 8B but doesn't have the 70B lag.

Moondream: The "tiny" vision model that actually worked when others crashed.

### Giving the Container "Eyes"

One of the most satisfying parts was getting a vision model to describe a local image. However, I ran into a classic container hurdle: the model couldn't "see" the files on my Rocky Linux host.

I had to recreate the Ollama container with a bind mount to my home directory:

```bash
# Mapping my local home to /images inside the container
-v /home/wsmoak:/files:ro
```

Even then, the heavy-hitter Llava crashed with a 500 Internal Server Error because it tried to grab too much VRAM for the image tensors. The solution was Moondream—a tiny, highly efficient vision model.

```
>>> Describe this image: /files/260122_2028.JPG
A black and brown cat is curled up in a cardboard box on a wooden floor... The background features a wooden floor and a white electrical outlet...
```

<a data-flickr-embed="true" href="https://www.flickr.com/photos/wsmoak/55107472252/in/dateposted-public/" title="moondream-cat-pic-260221"><img src="https://live.staticflickr.com/65535/55107472252_70ac425eff_z.jpg" width="640" height="174" alt="moondream-cat-pic-260221"/></a><script async src="//embedr.flickr.com/assets/client-code.js" charset="utf-8"></script>

## Open WebUI

While the terminal is great, a true "Local AI" experience needs a modern interface. I hooked up Open WebUI (running in its own Podman container) to my Ollama instance. This gives me a ChatGPT-like experience, but with 100% of the data staying on my local network.

<a data-flickr-embed="true" href="https://www.flickr.com/photos/wsmoak/55108615344/in/dateposted-public/" title="open-webui-cat-pic-260221"><img src="https://live.staticflickr.com/65535/55108615344_7fcf1a8986_z.jpg" width="640" height="552" alt="open-webui-cat-pic-260221"/></a><script async src="//embedr.flickr.com/assets/client-code.js" charset="utf-8"></script>

It correctly identified the cat's colors, its "comfortable" positioning, and even (sort of) spotted the electrical cord on the floor.

Initially, the UI would "hang" after I uploaded an image. Watching the Podman logs, I found that multiple models were fighting for the Titan RTX's 24GB. Because I had multiple models "warm," the GPU didn't have enough breathing room to process the image tokens quickly.

If your browser tab says something like "Your concise title here." this happens when Open WebUI tries to auto-generate a chat title in the background.

To fix the hangs and save resources, I made these changes:

OLLAMA_MAX_LOADED_MODELS=1 forces Ollama to unload the heavy coding model before starting the vision model. It adds a 10-second "swap" delay but prevents the UI from freezing.

I turned off auto-generation in the Open WebUI settings to save every bit of VRAM for the actual conversation.  I can add titles to the conversations I want to save.

Now, I can flip between summarizing text with an agent and analyzing photos of my cat without worrying about the system locking up.

### Lessons Learned

A smaller model running 100% on GPU is almost always a better user experience than a massive model splitting time with the CPU.

Keeping a separate SSH window open with watch -n 0.5 nvidia-smi is essential for understanding why a model is crashing or running slowly.

<a data-flickr-embed="true" href="https://www.flickr.com/photos/wsmoak/55108750965/in/dateposted-public/" title="nvidia-smi-260221_110309"><img src="https://live.staticflickr.com/65535/55108750965_b6be868ff7_z.jpg" width="640" height="298" alt="nvidia-smi-260221_110309"/></a><script async src="//embedr.flickr.com/assets/client-code.js" charset="utf-8"></script>

### References

* [Ollama Documentation](https://docs.ollama.com/)
* [Open WebUI Setup Guide](https://docs.openwebui.com/getting-started/quick-start/)
* [Rocky Linux Firewalld for Beginners](https://docs.rockylinux.org/10/guides/security/firewalld-beginners/)
* [Moondream AI](https://moondream.ai/)

### AI

I used Gemini to guide my exploration, troubleshoot problems, and to draft the structure of this post.

Copyright 2026 Wendy Smoak - This post first appeared on wsmoak.net and is CC BY-NC licensed.