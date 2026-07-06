# ArgoCD Challenge

This repository contains the complete two-part submission for an ArgoCD challenge

## AI use disclosure

For this task I used Claude code and Gemini.

For [task nr.1](task1), I created a draft design on my own; I used Gemini to investigate and validate the monitoring solution.

Once I had the overall design done, I used Claude code to help me generate the files (Taskfile, ArgoCD manifests etc). However I made sure that:

- I read every single line;

- I understood what it did and why it was there.

For [task nr.2](task2), I wanted to play with [`reveal js`](https://revealjs.com/) and [`mermaid js`](https://github.com/mermaid-js/mermaid), and I enrolled Claude code help to get a base scaffolding to build the slides, with the diagram, the slides transition and the coloring. I wrote the draft text and got help improving it.
The diagram itself is 100% Mermaid-generated; the CSS is purely cosmetic scaffolding around it.

## Access to the solutions

Clone the repo and `cd` into the root of the repo.

For [Task nr.1](task1), `cd task1; task -l`. More info are found in the [README.md](task1/README.md) inside the folder.

For [Task nr.2](task2), `cd task2`
