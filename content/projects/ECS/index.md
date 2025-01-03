---
title: "Entity-Compent-System"
date: 2024-10-30
draft: false
summary: "Entity-Component-System I build for our 6th gameproject at The Game Assembly."
categories: ["Post"]
tags: ["ECS","Entity-Component-System"]
---
{{< lead >}}
When life gives you lemons, make lemonade.
{{< /lead >}}
For our 6th game project I wrote my own Entity-Component-System.

{{< mermaid >}}
graph LR;
A[Entity]-->B[Component];
B-->C[Systems]
{{< /mermaid >}}

{{< figure
    src="AnimControllercallbacks.gif"
    >}}

