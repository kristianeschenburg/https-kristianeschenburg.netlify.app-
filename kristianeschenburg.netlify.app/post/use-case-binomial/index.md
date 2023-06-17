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
I'm building a model at work to predict amino acid identity at given protein sequence indices, given the local spatial and chemical environments of each amino acid.  I'm using a transformer-based geometric language model to do this.  A given protein is represeted as a graph object $G = (V,E)$, where $V$ are the nodes in the graph (here represented by individual amino acids), and $E$ the edges in the graph.  An edge exists between two amino acids if there are within a distance $d$ of one another.

Edges are characterized by a feature vector of length $F_{E}$ -- this feature vector includes information related to geometry, including Euclidean distance between the two amino acids, local orientation features, as well as rotational information.  Orientation and rotational information are computed using a local reference around each amino acid.

{{< figure src="ingraham.png" title="" lightbox="true" >}}

This model is trained in a similar manner to a masked language model.  Given a set of training graphs, we randomly "hide" amino acids in each training graph, and ask the model to fill in these hidden amino acids, given signals in the local environment.  We're not providing any information to the model, other than the geometric information described above.  The model is unaware of the charge, size, shape, etc. of any amino acid -- we're hypothesizing that a graph-based model should be able to learn these properties implicity to optimize model performance.

