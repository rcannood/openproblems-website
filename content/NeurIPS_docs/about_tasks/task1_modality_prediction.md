---
title: "Task 1: Modality Prediction"
linktitle: "Task 1: Modality Prediction"
type: book
date: "2021-08-02T00:00:00+01:00"
# Prev/next pager order (if `docs_section_pager` enabled in `params.toml`)
weight: 2
---
## Predicting the flow of information from DNA to RNA and RNA to Protein
Experimental techniques to measure multiple modalities within the same single cell are increasingly becoming available. The demand for these measurements is driven by the promise to provide a deeper insight into the state of a cell. Yet, the modalities are also intrinsically linked. We know that DNA must be accessible (ATAC data) to produce mRNA (expression data), and mRNA in turn is used as a template to produce protein (protein abundance). These processes are regulated often by the same molecules that they produce: for example, a protein may bind DNA to prevent the production of more mRNA. Understanding these regulatory processes would be transformative for synthetic biology and drug target discovery. Any method that can predict a modality from another must have accounted for these regulatory processes, but the demand for multi-modal data shows that this is not trivial.

<figure>
  <img src="/media/tasks/predict.svg">
  <figcaption>
    <h3>
      Task 1: Prediction
    </h3>
    <p style="font-size: medium;">
      In this task, the goal is to take one modality (ATAC or RNA) and predict the other modality (RNA or Protein) for all features in each cell (only ATAC + RNA shown). Performance is measured using Root Mean Square Error on log1p size-factor normalized counts.
    </p>
  </figcaption>
</figure>

This task requires translating information between multiple layers of gene regulation. In some ways, this is similar to the task of machine translation. In machine translation, the same sentiment is expressed in multiple languages and the goal is to train a model to represent the same meaning in a different language. In this context, the same cellular state is measured in two different feature sets and the goal of this task is to translate the information about cellular state from one modality to the other. 

## Task API

The following section describes the task API for the Modality Prediction task. Competitors must submit their code as a Viash component. To facilitate creation of these components, [starter kits](//neurips_docs/submission/starter_kit_contents) have been provided.

### Input data formats

{{% callout note  %}}
The data format and attributes provided in the input data is tailored to each task and may differ from the publicly released [benchmarking dataset](/neurips_docs/data/dataset/).  **Only the attributes listed in the following section will be accessible to methods submitted to the competition.**
{{% /callout  %}}

#### Inputs to methods

Method components should expect three inputs, `--input_train_mod1`, `--input_train_mod2`, and `--input_test_mod1`. They are all paths to [AnnData](https://anndata.readthedocs.io/en/latest/) h5ad files with the attributes below. More information can be found on AnnData objects [here](/neurips_docs/submission/quickstart/). `mod1` and `mod2` refer to the modality of the datasets as defined by `feature_type`.  One file will always have `feature_type` be `"GEX"` and the other will be `"ATAC"` or `"ADT"`. For the purposes of the competition, components should expect the following 4 combinations of modalities:

| `mod1`   | `mod2`   |
|----------|----------|
| `"GEX"`  | `"ATAC"` |
| `"ATAC"` | `"GEX"`  |
| `"GEX"`  | `"ADT"`  |
| `"ADT"`  | `"GEX"`  |


#### Objective for methods

Submission components must predict `mod2` for the cells provided in `--input_test_mod1`. For methods that do not involve pre-trained models, training data is also provided in the `--input_train_mod[1|2]` files. For methods that involve pre-trained models, these training datasets can be ignored.

{{% callout note  %}}
Note, you do not need to return predictions for all four combinations of inputs and outputs. We will be independently ranking and awarding prizes to each combination as described below in [Prizes](#prizes). For more details, see the [FAQs](/neurips_docs/submission/faq/#what-if-i-only-want-to-compete-for-one-of-the-prizes-in-a-task)
{{% /callout  %}}


#### Attributes of input data

The input data objects have the following attributes:


```plaintext
adata
  Input AnnData object for modality 1 or 2

  Attributes
  ----------
  adata.X : ndarray, shape=(n_obs, n_var)
    Sparse profile matrix of given modality. If .var['feature_types'] == "GEX" or "ADT",
    values in adata.X represent expression counts for each gene. If
    .var['feature_types'] == "ATAC", values represent counts of reads in peaks for
    chromatin accessibility
  adata.uns['dataset_id'] : str
    The name of the dataset.
  adata.obs["batch"] : ndarray, shape=(n_obs,)
    The batch from which the data was sequenced. Has format "s[1-4]d[1-9]" indicating the site and
    donor associated with the batch.
  adata.obs_names : ndarray, shape=(n_obs,)
    Ids for the cells.
  adata.var['feature_types']: ndarray, shape=(n_var,)
    The modality of this file, should be equal to "GEX", "ATAC" or "ADT".
  adata.var_names : ndarray, shape=(n_var,)
    Ids for the features.
```

Examples of how to load and process the data are contained in the [starter kits](/neurips_docs/submission/starter_kit_contents) for the respective programming language.

#### Normalization and transformation of data for the prediction task

To make the task more straightforward, we have followed common practices for normalizing and transforming data of each modality. The raw data is also provided in `adata.layers` as described below. Please note, **the performance metric will be calculated on the normalized and transformed data stored in** `adata.X` **for each of the modality types below.**

For full details on preprocessing, see the [Data Preprocessing](/neurips_docs/data/dataset/#preprocessing) notes.

**GEX**  

For this task, gene expression data stored in `adata.X` for the training and test data has been size-factor normalized and log1p transformed.  Raw UMI counts are available in `adata.layers["counts"]`. Size factors are accessible in `adata.obs["size_factors"]`

**ATAC**

For this task, ATAC data stored in `adata.X` for the training and test data has been binarized and subset to 10000 random peaks. The raw UMI counts for each peak can be found in `adata.layers["counts"]`.

**ADT**

For this task, ADT derived protein abundance measures have been centered log-ration (CLR) normalized. Raw ADT counts can be found in `adata.layers["counts"]`.

### Output data formats

This component should output only one h5ad file whose path is specified via `--output`, containing the predicted profile values of modality 2 for the **test cells only**. It must have the following attributes:

```plaintext
adata
  Output AnnData object containing predictions for modality 2 in the "test" cells

  Attributes
  ----------
  adata.X : ndarray, shape=(n_obs, n_var)
    Sparse profile matrix.
  adata.uns['dataset_id'] : str
    The name of the dataset.
  adata.uns['method_id'] : str
    The name of the prediction method. This is used to track submissions.
  adata.var['feature_types'] : ndarray, shape=(n_var,)
    The modality of this file, should be equal to "GEX", "ATAC" or "ADT".
  adata.obs_names : ndarray, shape=(n_obs,)
    Ids for the cells.
```

### Metric

Performance in task 1 is measured using the root mean squared error between the observed and predicted values for modality 2 in the `test` set. Lower values are better.

The metric function used to evaluate the prediction has the following structure (this example employs `Python` syntax; the `R` evaluation function is functionally equivalent):

```plaintext
def calculate_rmse(adata_mod2, adata_mod2_answer):  
    '''Function to calculate MSE between prediction and solution for the test sets  

    Params
    ------
    adata_mod2 : AnnData, shape=(n_obs, n_var)
      User-submitted prediction for expression of mod2 in cells from the test set
    adata_mod2_answer : AnnData, shape=(n_obs, n_var)
      Measured values for expression of mod2 in the test set

    Returns
    -------
    mean_square_error : float
      The mean squared error between the predicted and observed values for all features in
      the test set.
    '''
    from sklearn.metrics import mean_square_error
    return mean_square_error(adata_mod2.X, adata_mod2_answer.X, squared=False)
```

## Prizes

For this task, five prizes of $1000 will be awarded to the submissions for each of the following criteria:
1. Best performance predicting GEX → ATAC
2. Best performance predicting ATAC → GEX
2. Best performance predicting GEX → ADT
2. Best performance predicting ADT → GEX
3. Best performance on average across modalities

[Terms and Conditions](/neurips_docs/submission/terms/) apply.
