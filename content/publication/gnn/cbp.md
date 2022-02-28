---
title: "Learning Cortical Parcellations Using Graph Neural Networks"
authors:
- Kristian M. Eschenburg
- Tom Grabowski
- David Haynor
date: "2021-12-24"
doi: ""

# Schedule page publish date (NOT publication's date).
publishDate: "2021-12-24"

# Publication type.
# Legend: 0 = Uncategorized; 1 = Conference paper; 2 = Journal article;
# 3 = Preprint / Working Paper; 4 = Report; 5 = Book; 6 = Book section;
# 7 = Thesis; 8 = Patent
publication_types: ["2"]

# Publication name and optional abbreviated publication name.
publication: ""
publication_short: ""

abstract: "Deep learning has been applied to magnetic resonance imaging (MRI) for a variety of purposes, ranging from the acceleration of image acquisition and image denoising to tissue segmentation and disease diagnosis. Convolutional neural networks have been particularly useful for analyzing MRI data due to the regularly sampled spatial and temporal nature of the data. However, advances in the field of brain imaging have led to network- and surface-based analyses that are often better represented in the graph domain. In this analysis, we propose a general purpose cortical segmentation method that, given resting-state connectivity features readily computed during conventional MRI pre-processing and a set of corresponding training labels, can generate cortical parcellations for new MRI data. We applied recent advances in the field of graph neural networks to the problem of cortical surface segmentation, using resting-state connectivity to learn discrete maps of the human neocortex. We found that graph neural networks accurately learn low-dimensional representations of functional brain connectivity that can be naturally extended to map the cortices of new datasets. After optimizing over algorithm type, network architecture, and training features, our approach yielded mean classification accuracies of 79.91% relative to a previously published parcellation. We describe how some hyperparameter choices including training and testing data duration, network architecture, and algorithm choice affect model performance."

# Summary. An optional shortened abstract.
summary: We examine the utility of graph neural networks for the purpose of learning cortical segmentations.  We show that attention-based transformer networks significantly outperform conventional GCN and linear feed-forward variants for the purpose of generating accurate reproducible cortical maps.

tags:
- graph neural networks
- deep learning
- image segmentation
- supervised machine learning
featured: true

links:
- name: Paper
  url: https://www.frontiersin.org/articles/10.3389/fnins.2021.797500/full
url_pdf: ''
url_code: ''
url_dataset: ''
url_poster: ''
url_project: ''
url_slides: ''
url_source: ''
url_video: ''

# Featured image
# To use, add an image named `featured.jpg/png` to your page's folder. 
image:
  caption: ""
  focal_point: ""
  preview_only: false

# Associated Projects (optional).
#   Associate this publication with one or more of your projects.
#   Simply enter your project's folder or file name without extension.
#   E.g. `internal-project` references `content/project/internal-project/index.md`.
#   Otherwise, set `projects: []`.
# projects:
# - internal-project

# Slides (optional).
#   Associate this publication with Markdown slides.
#   Simply enter your slide deck's filename without extension.
#   E.g. `slides: "example"` references `content/slides/example/index.md`.
#   Otherwise, set `slides: ""`.
# slides: example
---