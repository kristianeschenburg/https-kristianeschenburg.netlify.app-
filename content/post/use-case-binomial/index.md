---
# Documentation: https://sourcethemes.com/academic/docs/managing-content/

title: "Use-case: de-novo protein design"
subtitle: "bonomial masking fractions"
summary: ""
authors: []
tags: []
categories: []
date: 2023-04-20T12:43:32-07:00
lastmod: 2023-04-20T12:43:32-07:00
featured: false
draft: true

# Featured image
# To use, add an image named `featured.jpg/png` to your page's folder.
# Focal points: Smart, Center, TopLeft, Top, TopRight, Left, Right, BottomLeft, Bottom, BottomRight.
image:
  caption: ""
  focal_point: ""
  preview_only: false

# Projects (optional).
#   Associate this post with one or more of your projects.
#   Simply enter your project's folder or file name without extension.
#   E.g. `projects = ["internal-project"]` references `content/project/deep-learning/index.md`.
#   Otherwise, set `projects = []`.
projects: []
---
I'm building a model to predict amino acid identity at given protein sequence indices, given the local spatial and chemical environments of each amino acid.  I'm using a transformer-based geometric language model to do this.  A given protein is represeted as a graph object $G = (V,E)$, where $V$ are the nodes in the graph (here represented by individual amino acids), and $E$ the edges in the graph.  An edge exists between two amino acids if there are within a distance $d$ of one another.

Edges are characterized by a feature vector of length $F_{E}$ -- this feature vector includes information related to geometry, including Euclidean distance between the two amino acids, local orientation features, as well as rotational information.  Orientation and rotational information are computed using a local reference around each amino acid.

{{< figure src="ingraham.png" title="Encoder-decoder model for de-novo protein design.  [Ingraham et al. 2019](https://papers.nips.cc/paper_files/paper/2019/hash/f3a4ff4839c56a5f460c88cce3666a2b-Abstract.html)." lightbox="true" >}}

This model is trained in a similar manner to a masked language model.  Given a set of training graphs (analogous to sequences), we randomly "hide" amino acids in each training graph, and ask the model to fill in these hidden amino acids, given signals in the local environment.  In large language models, these tokens are filled in sequentially, based on token index values, $\\{a,b,c\\}$ -- for protein design, it has been shown that a randomly permutated decoding order $\\{a,b,c...z\\}_{\pi}$ is preferential, and helps the models generalize better during the design stage.  We're not providing any information to the model, other than the geometric information described above.  The model is unaware of any *chemical* properties, such as charge, size, shape, polarity, etc. of any amino acid -- we're hypothesizing that a successful protein design model should be able to implicitly learn these properties during optimization.

One of the parameters in our model is what we're calling the "masking fraction" e.g. how many tokens are "hidden" from the model, or, how many tokens the model is asked to predict.