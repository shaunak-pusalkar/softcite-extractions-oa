# Softcite Extractions from Open Access Literature

The softcite-extractions-oa dataset is a collection of ML-identified mentions of software detected in about 24 million academic papers. The papers are all open access papers available circa 2024. The extractions were created from academic PDFs using the [Softcite mention extraction toolchain](https://github.com/softcite#mention-extraction-tool-chain), which is built on the [Grobid](https://github.com/kermitt2/grobid) model trained on the [Softcite Annotations dataset v2](https://github.com/softcite/softcite_dataset_v2).  More details available at the [Softcite Org home page](https://github.com/softcite/). 

This work used JetStream 2 at Indiana through allocation CIS220172 from the Advanced Cyberinfrastructure Coordination Ecosystem: Services & Support (ACCESS) program, which is supported by National Science Foundation grants #2138259, #2138286, #2138307, #2137603, and #2138296.

## The data

The data files are hosted outside github, on Zenodo at <https://zenodo.org/records/15149379>.  This GitHub repo is documentation and hosts the files used to convert the data from json into tabular format in parquet.

### Reporting extraction errors/omissions 

These extractions are the result of a machine learning model; they are probabilistic and will have both false positives and false negatives. The performance of the model shows f-scores around 0.8 (see <https://doi.org/10.1145/3459637.3481936> for full details, and <https://github.com/softcite/#papers> for more details, including the annotation scheme used in the underlying gold standard dataset. See <https://github.com/softcite/> for the training data, models, tools, and papers.

Please create [Issues](https://github.com/softcite/softcite-extractions-oa/issues) in this repository when you encounter problems in the dataset. That said, we can't correct these manually, but any explanation you can give will help us improve the training data and improve the model. Please also share transformations you have to apply to the dataset in your work with it.

## The data model

A __paper__ can contain many __mentions__, each of which was found in a full-text snippet of __context__, and extracts the (raw and normalized) __software name__ , __version number__, __creator__, __url__, as well as associated __citation__ to the reference list of the paper.

Each __mention__ has multiple __purpose assessments__ about the relationship between the software and the paper: Was the software __used__ in the research?, Was it __created__ in the course of the research?, Was the software __shared__ alongside this paper? These probabilistic assessments (0..1 range) are made in two ways: using only the information from the specific mention and using all the mentions within a single paper together (mention-level vs document-level); thus each mention has six __purpose assessments__.

<img src="class-diagram.png" alt="drawing" width="300"/>

## Getting Started

### Getting the Parquet files

Parquet files are available from Zenodo at <https://zenodo.org/records/15149379>.  There are three sub-folders:

```
full_dataset
p01_one_percent_random_subset
p05_five_percent_random_subset
```

The random subsets are subsets of papers, with all of the extractions in those papers. We created these to make prototyping analyses easier.  Inside each folder are three files:

```
papers.parquet
mentions.pdf.parquet
purpose_assessments.pdf.parquet
```
### Example Analyses

For these examples, the 5% subset of the data is used. First 3 examples are shown in R and then the same 3 examples are shown in Python.

Examples in R
These examples require the `tidyverse` and `arrow` packages to run, but should otherwise work as-is.

```R
library(tidyverse)
library(arrow)
```

1. How many papers mention OpenStreetMap?

This example filters by `software_normalied` as this is less noisy than `software_raw`.

```R
> mentions <- open_dataset("p05_five_percent_random_subset/mentions.pdf.parquet")
> mentions |>
+   filter(software_normalized == "OpenStreetMap") |>
+   select(paper_id) |>
+   distinct() |>
+   count() |>
+   collect()
# A tibble: 1 × 1
      n
  <int>
1   376
```


2. How did the number of papers referencing STATA each year change from 2000-2020?

By joining the Mentions table with Papers, we can compute statistics requiring access to paper metadata. Analyses like these are why we include fields such as `paper_id` in Mentions, even though it denormalizes the tables.

```R
> papers <- open_dataset("p05_five_percent_random_subset/papers.parquet")
> mentions <- open_dataset("p05_five_percent_random_subset/mentions.pdf.parquet")
> 
> mentions |>
+   filter(software_normalized == "STATA") |>
+   select(paper_id) |>
+   distinct() |>
+   inner_join(papers, by = c("paper_id")) |>
+   filter(published_year >= 2000, published_year <= 2020) |>
+   count(published_year) |>
+   arrange(published_year) |>
+   collect()
# A tibble: 21 × 2
   published_year     n
            <int> <int>
 1           2000    11
 2           2001    14
 3           2002    20
 4           2003    29
 5           2004    51
 6           2005    32
 7           2006    42
 8           2007    49
 9           2008    77
10           2009    87
# ℹ 11 more rows
# ℹ Use `print(n = ...)` to see more rows
```

3. What are the most popular software packages used since 2020, by number of distinct papers?

Answering this question requires joining all three tables.
Especially with the full dataset, we generally recommend using `select` statements before and after joins to reduce memory overhead.
Here we use the PurposeAssessments table to evaluate whether software was "used" in a paper.
The "document" scope is appropriate here as we're interested in whether the software was used by the paper, not whether particular mentions of the software indicate this.

```R
> papers <- open_dataset("p05_five_percent_random_subset/papers.parquet")
> mentions <- open_dataset("p05_five_percent_random_subset/mentions.pdf.parquet")
> purposes <- open_dataset("p05_five_percent_random_subset/purpose_assessments.pdf.parquet")
> 
> papers |>
+   filter(published_year >= 2020) |>
+   select(paper_id) |>
+   inner_join(mentions, by=c("paper_id")) |>
+   select(software_mention_id, software_normalized) |>
+   inner_join(purposes, by=c("software_mention_id")) |>
+   filter(scope=="document", purpose=="used", certainty_score > 0.5) |>
+   select(paper_id, software_normalized) |>
+   distinct() |>
+   count(software_normalized) |>
+   arrange(desc(n)) |>
+   collect()
# A tibble: 79,730 × 2
   software_normalized     n
   <chr>               <int>
 1 SPSS                22596
 2 GraphPad Prism       8080
 3 Excel                6131
 4 ImageJ               5477
 5 MATLAB               5117
 6 SAS                  3480
 7 SPSS Statistics      3065
 8 Stata                2545
 9 script               2247
10 Matlab               2225
# ℹ 79,720 more rows
# ℹ Use `print(n = ...)` to see more rows
```
Examples in Python

Setup & Load Libraries and load data

```Python
import pandas as pd
# Choose which dataset folder to use

# Comment this to switch datasets

#dataset_choice = "full_dataset"
#dataset_choice = "p01_one_percent_random_subset"
dataset_choice = "p05_five_percent_random_subset"


# Construct the base path dynamically
base_path = f"/content/drive/MyDrive/{dataset_choice}"

# Load Parquet files
papers = pd.read_parquet(f"{base_path}/papers.parquet")
mentions = pd.read_parquet(f"{base_path}/mentions.pdf.parquet")
purposes = pd.read_parquet(f"{base_path}/purpose_assessments.pdf.parquet")

# Quick peek
print("Using dataset:", dataset_choice)
print("Papers:", papers.shape)
print("Mentions:", mentions.shape)
print("Purposes:", purposes.shape)
```

1. How many papers mention OpenStreetMap?

This example filters by `software_normalied` as this is less noisy than `software_raw`.

```Python
openstreetmap_papers = (
    mentions[mentions["software_normalized"] == "OpenStreetMap"]
    .drop_duplicates(subset="paper_id")   # distinct paper_id
)

count = openstreetmap_papers["paper_id"].nunique()
print("Number of papers mentioning OpenStreetMap:", count)

#Number of papers mentioning OpenStreetMap: 376
```

2. How did the number of papers referencing STATA each year change from 2000-2020?

By joining the Mentions table with Papers, we can compute statistics requiring access to paper metadata. Analyses like these are why we include fields such as `paper_id` in Mentions, even though it denormalizes the tables.

```Python
stata_mentions = (
    mentions[mentions["software_normalized"] == "STATA"]
    [["paper_id"]].drop_duplicates()
)

stata_by_year = (
    stata_mentions.merge(papers, on="paper_id", how="inner")
    .query("2000 <= published_year <= 2020")
    .groupby("published_year")
    .size()
    .reset_index(name="n")
    .sort_values("published_year")
)

print(stata_by_year)

    published_year    n
0             2000   11
1             2001   14
2             2002   20
3             2003   29
4             2004   51
5             2005   32
6             2006   42
7             2007   49
8             2008   77
9             2009   87
10            2010  107
11            2011  122
12            2012  139
13            2013  172
14            2014  169
15            2015  208
16            2016  268
17            2017  291
18            2018  310
19            2019  382
20            2020  485

```

3. What are the most popular software packages used since 2020, by number of distinct papers?

Answering this question requires joining all three tables.
Especially with the full dataset, we generally recommend using `select` statements before and after joins to reduce memory overhead.
Here we use the PurposeAssessments table to evaluate whether software was "used" in a paper.
The "document" scope is appropriate here as we're interested in whether the software was used by the paper, not whether particular mentions of the software indicate this.

```Python

popular_software = (
    papers.query("published_year >= 2020")[["paper_id"]]
    .merge(mentions[["software_mention_id", "software_normalized", "paper_id"]],
           on="paper_id", how="inner")
    .merge(purposes[["software_mention_id", "scope", "purpose", "certainty_score"]],
           on="software_mention_id", how="inner")
    .query("scope == 'document' and purpose == 'used' and certainty_score > 0.5")
    [["paper_id", "software_normalized"]]
    .drop_duplicates()
    .groupby("software_normalized")
    .size()
    .reset_index(name="n")
    .sort_values("n", ascending=False)
)

print(popular_software.head(10))  # show top 10

      software_normalized      n
53354                SPSS  22596
25325      GraphPad Prism   8080
18712               Excel   6131
28455              ImageJ   5477
33089              MATLAB   5117
51852                 SAS   3480
53497     SPSS Statistics   3065
58191               Stata   2545
77218              script   2247
35936              Matlab   2225

```

## Additional details and provenance

The Grobid extraction pipeline worked with multiple sources for each paper, including PDFs and xml sources from publishers, such as JATS and TEI XML.  This produced json files, which were then processed to tabular formats in parquet. 

The tablular dataset includes only extractions from PDF sources, to avoid complexity of multiple source types for a single paper. This decision was made easier based on the reality that PDF was available for all papers, but other papers sources were only available for smaller subsets.  

Details of the full json data, from all source document types, and the way those were read and mapped to tabular data are available in [Extracting Tables](EXTRACTING_TABLES.md).
