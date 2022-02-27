---
# Documentation: https://sourcethemes.com/academic/docs/managing-content/

title: "Visualizing SQL Schemas"
subtitle: ""
summary: ""
authors: []
tags: []
categories: []
date:   2022-02-22T10:24:17-07:00
lastmod: 2022-02-22T10:24:17-07:00
featured: false
draft: false

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

I was recently tasked with examining databases related to some computer vision tools that my company had acquired.  Basically, the framework was as follows... Clients/users would sign up for some service with the goal in mind of building a model to classify a set of microscopy images.  These models could then be used by the client for downstream services.  The users interacted with this tool via through a WebApp, through which they could upload the relevant training and validation data, as well as the full suite of models that they had previously trained.  The software used AWS EC2 instances to train and test their models, and a database to store all the relevant metadata associated with the users and experiments.

However, I was not provided with any relevant information about the database schema, rendering interpretation of the API and results difficult.  Given that I had access to the MySQL database, I wanted to be able to visualize the interactions between all the relevant tables.

Assuming that `mysql` and `MySQL Workbench` are installed, and that you know the database name and corresponding account password, you can run the following command to export the database to an SQL file.  The `-u`, `-p`, and `--no-data` options correspond to the user ID, password, and desire (or lack thereof) to export the data entries as well.

```bash
mysqldump -u root -p --no-data ${DB_NAME} > ${OUTPUT_NAME}.sql
```

You can then reverse engineer the database schema by clicking `Database -> Reverse Engineer` and subsequent the steps:

{{< figure src="reverse.png" title="" lightbox="true" >}}

We can then actually visualize the database schema and table structures in `MySQL Workbench`.  Below, we see that all SQL tables are joined to the table called `users` via their corresponding `user_id` field.  At a minimum, we'll be able to sort experiments by `user_id` -- however, I'm hoping there is also a join for `model_name` or something along those lines, so that we can more easily interrogate the specific parameterizations of each model.

{{< figure src="schema.png" title="" lightbox="true" >}}