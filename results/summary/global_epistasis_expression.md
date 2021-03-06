# Fit global epistasis models to DMS expression data

The [`dms_variants.globalepistasis`](https://jbloomlab.github.io/dms_variants/dms_variants.globalepistasis) module is based off the "global epistasis" concepts of [Otwinoski et
al](https://doi.org/10.1073/pnas.1804015115) and [Sailer and
Harms](https://doi.org/10.1534/genetics.116.195214) -- that there exists an underlying latent phenotype that mutations affect additively, and
then an observed (measured) phenotype that is a non-linear
function of the latent phenotype.


## Setup for analysis

Import Python modules / packages:


```python
import collections
import os
import itertools
import random
import tempfile
import time
import warnings

import pandas as pd

from plotnine import *

import dms_variants.binarymap
import dms_variants.codonvarianttable
import dms_variants.globalepistasis
import dms_variants.plotnine_themes
import dms_variants.simulate
from dms_variants.constants import CBPALETTE, CODONS_NOSTOP

import yaml
```

Set the gray grid [plotnine theme](https://plotnine.readthedocs.io/en/stable/generated/plotnine.themes.theme.html?highlight=themes) defined in [dms_variants.plotnine_themes](https://jbloomlab.github.io/dms_variants/dms_variants.plotnine_themes.html):


```python
theme_set(dms_variants.plotnine_themes.theme_graygrid())
```

Versions of key software:


```python
print(f"Using dms_variants version {dms_variants.__version__}")
```

    Using dms_variants version 0.6.0


Set pandas display options to show large chunks of Data Frames in this
example:


```python
pd.set_option('display.max_columns', 30)
pd.set_option('display.width', 500)
```

Hide warnings that clutter output:


```python
warnings.simplefilter('ignore')
```

### Parameters for notebook
Read the configuration file:


```python
with open('config.yaml') as f:
    config = yaml.safe_load(f)
```

Make output directory if needed:


```python
os.makedirs(config['global_epistasis_expr_dir'], exist_ok=True)
os.makedirs(config['figs_dir'], exist_ok=True)
```

## Read in Sort-seq expression mean fluorescence scores by `barcode`
Read in Sort-seq expression measurements. I will first fit global epistasis models using the ML_meanF measurements. Rename meanF column to be func_score, and var column to func_score_var. Remove rows with NaN for func_score. For empty aa_substitutions (wildtype), it is being replaced with NA. Make back to an empty string.


```python
df = pd.read_csv(config['expression_sortseq_file'])
df.rename(columns={'delta_ML_meanF':'func_score','var_ML_meanF':'func_score_var'},inplace=True)
func_scores = df[pd.notnull(df['func_score'])]
func_scores.fillna('',inplace=True)
func_scores.head()
```




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>library</th>
      <th>target</th>
      <th>barcode</th>
      <th>variant_call_support</th>
      <th>total_count</th>
      <th>ML_meanF</th>
      <th>func_score</th>
      <th>func_score_var</th>
      <th>variant_class</th>
      <th>aa_substitutions</th>
      <th>n_aa_substitutions</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>1</th>
      <td>lib1</td>
      <td>SARS-CoV-2</td>
      <td>AAAAAAAAACTTAAAT</td>
      <td>2</td>
      <td>64.71</td>
      <td>7.45</td>
      <td>-3.01</td>
      <td>0.04</td>
      <td>&gt;1 nonsynonymous</td>
      <td>N13S L60P K94N S147T C150Y</td>
      <td>5</td>
    </tr>
    <tr>
      <th>2</th>
      <td>lib1</td>
      <td>SARS-CoV-2</td>
      <td>AAAAAAAAACTTCAAT</td>
      <td>5</td>
      <td>117.96</td>
      <td>7.92</td>
      <td>-2.54</td>
      <td>0.03</td>
      <td>&gt;1 nonsynonymous</td>
      <td>A22C R127G E141D L188V</td>
      <td>4</td>
    </tr>
    <tr>
      <th>3</th>
      <td>lib1</td>
      <td>SARS-CoV-2</td>
      <td>AAAAAAAACAAGCAGA</td>
      <td>6</td>
      <td>244.34</td>
      <td>8.93</td>
      <td>-1.53</td>
      <td>0.01</td>
      <td>1 nonsynonymous</td>
      <td>N13F</td>
      <td>1</td>
    </tr>
    <tr>
      <th>4</th>
      <td>lib1</td>
      <td>SARS-CoV-2</td>
      <td>AAAAAAAACAATATAA</td>
      <td>1</td>
      <td>95.35</td>
      <td>6.21</td>
      <td>-4.25</td>
      <td>0.03</td>
      <td>&gt;1 nonsynonymous</td>
      <td>C6K T15W K94Y V103W</td>
      <td>4</td>
    </tr>
    <tr>
      <th>5</th>
      <td>lib1</td>
      <td>SARS-CoV-2</td>
      <td>AAAAAAAACAGGTTGC</td>
      <td>4</td>
      <td>212.43</td>
      <td>7.73</td>
      <td>-2.73</td>
      <td>0.02</td>
      <td>&gt;1 nonsynonymous</td>
      <td>V71K P149L N157T</td>
      <td>3</td>
    </tr>
  </tbody>
</table>
</div>



## Fit global epistasis models to ML meanF
We now fit global epistasis models to the functional scores.
For background on these models, see [Otwinoski et al (2018)](https://www.pnas.org/content/115/32/E7550).
The models we fits are implemented in [dms_variants.globalepistasis](https://jbloomlab.github.io/dms_variants/dms_variants.globalepistasis.html), which has extensive documentation.

The primary model of interest is [MonotonicSplineEpistasis](https://jbloomlab.github.io/dms_variants/dms_variants.globalepistasis.html#dms_variants.globalepistasis.MonotonicSplineEpistasis), which assumes that the observed phenotype is a monotonic non-linear function of an underlying additive latent phenotype.
As a control, we also fit a [NoEpistasis](https://jbloomlab.github.io/dms_variants/dms_variants.globalepistasis.html#dms_variants.globalepistasis.NoEpistasis) model, which assumes that mutations simply contribute additively to the observed phenotype:

For the fitting, we first convert the set of functional scores into a binary representation using a [BinaryMap]((https://jbloomlab.github.io/dms_variants/dms_variants.binarymap.html#dms_variants.binarymap.BinaryMap)).
Then we create the model, fit it, and store it.


```python
# NBVAL_IGNORE_OUTPUT

models = {}  # store models, keyed by `(epistasistype, likelihoodtype, sample, lib)`

for (lib), scores in func_scores.groupby(['library']):
   
    bmap = dms_variants.binarymap.BinaryMap(scores)
    
    for epistasistype, likelihoodtype, Model in [
            ('global epistasis', 'Gaussian', dms_variants.globalepistasis.MonotonicSplineEpistasisGaussianLikelihood),
            ('no epistasis', 'Gaussian', dms_variants.globalepistasis.NoEpistasisGaussianLikelihood),
            ('global epistasis', 'Cauchy', dms_variants.globalepistasis.MonotonicSplineEpistasisCauchyLikelihood),
            ('no epistasis', 'Cauchy', dms_variants.globalepistasis.NoEpistasisCauchyLikelihood),
            ]:
        print(f"Fitting {epistasistype} with {likelihoodtype} likelihood model to {lib}...", end=' ')
    
        start = time.time()
        model = Model(bmap)
        model.fit()  # do NOT change ftol in normal use, this is just for test
        print(f"fitting took {time.time() - start:.1f} sec.")
        models[(epistasistype, likelihoodtype, lib)] = model
```

    Fitting global epistasis with Gaussian likelihood model to lib1... fitting took 152.3 sec.
    Fitting no epistasis with Gaussian likelihood model to lib1... fitting took 0.5 sec.
    Fitting global epistasis with Cauchy likelihood model to lib1... fitting took 412.6 sec.
    Fitting no epistasis with Cauchy likelihood model to lib1... fitting took 5.1 sec.
    Fitting global epistasis with Gaussian likelihood model to lib2... fitting took 128.6 sec.
    Fitting no epistasis with Gaussian likelihood model to lib2... fitting took 0.4 sec.
    Fitting global epistasis with Cauchy likelihood model to lib2... fitting took 244.6 sec.
    Fitting no epistasis with Cauchy likelihood model to lib2... fitting took 3.3 sec.


Now we want to see which model fits the data better.
To do this, we get the log likelihood of each model along with the number of model parameters and use it to calculate the [AIC](https://en.wikipedia.org/wiki/Akaike_information_criterion).
Models with lower AIC are better, and below we see that the [MonotonicSplineEpistasis](https://jbloomlab.github.io/dms_variants/dms_variants.globalepistasis.html#dms_variants.globalepistasis.MonotonicSplineEpistasis) global epistasis model always fits the data much better:


```python
# NBVAL_IGNORE_OUTPUT

logliks_df = (
    pd.DataFrame.from_records(
            [(epistasistype, likelihoodtype, lib, model.nparams, model.loglik) for
             (epistasistype, likelihoodtype, lib), model in models.items()],
            columns=['model', 'likelihood type', 'library',
                     'n_parameters', 'log_likelihood']
            )
    .assign(AIC=lambda x: 2 * x['n_parameters'] - 2 * x['log_likelihood'])
    .set_index(['library'])
    )

logliks_df.round(1)
```




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>model</th>
      <th>likelihood type</th>
      <th>n_parameters</th>
      <th>log_likelihood</th>
      <th>AIC</th>
    </tr>
    <tr>
      <th>library</th>
      <th></th>
      <th></th>
      <th></th>
      <th></th>
      <th></th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>lib1</th>
      <td>global epistasis</td>
      <td>Gaussian</td>
      <td>4003</td>
      <td>-61437.0</td>
      <td>130880.1</td>
    </tr>
    <tr>
      <th>lib1</th>
      <td>no epistasis</td>
      <td>Gaussian</td>
      <td>3997</td>
      <td>-106496.8</td>
      <td>220987.5</td>
    </tr>
    <tr>
      <th>lib1</th>
      <td>global epistasis</td>
      <td>Cauchy</td>
      <td>4003</td>
      <td>-44897.1</td>
      <td>97800.1</td>
    </tr>
    <tr>
      <th>lib1</th>
      <td>no epistasis</td>
      <td>Cauchy</td>
      <td>3997</td>
      <td>-117039.0</td>
      <td>242072.1</td>
    </tr>
    <tr>
      <th>lib2</th>
      <td>global epistasis</td>
      <td>Gaussian</td>
      <td>4002</td>
      <td>-58077.8</td>
      <td>124159.6</td>
    </tr>
    <tr>
      <th>lib2</th>
      <td>no epistasis</td>
      <td>Gaussian</td>
      <td>3996</td>
      <td>-98805.6</td>
      <td>205603.2</td>
    </tr>
    <tr>
      <th>lib2</th>
      <td>global epistasis</td>
      <td>Cauchy</td>
      <td>4002</td>
      <td>-41108.7</td>
      <td>90221.4</td>
    </tr>
    <tr>
      <th>lib2</th>
      <td>no epistasis</td>
      <td>Cauchy</td>
      <td>3996</td>
      <td>-108540.5</td>
      <td>225072.9</td>
    </tr>
  </tbody>
</table>
</div>




```python
# check to confirm the global epistasis model is better in all cases
assert (logliks_df
        .reset_index()
        .pivot_table(index=['library'],
                     values='AIC',
                     columns='model'
                     )
        .assign(global_better=lambda x: x['global epistasis'] < x['no epistasis'])
        ['global_better']
        .all()
        )
```

Because the [MonotonicSplineEpistasis](https://jbloomlab.github.io/dms_variants/dms_variants.globalepistasis.html#dms_variants.globalepistasis.MonotonicSplineEpistasis) global epistasis models fit so much better than the additive [NoEpistasis](https://jbloomlab.github.io/dms_variants/dms_variants.globalepistasis.html#dms_variants.globalepistasis.NoEpistasis) models, below we are just going to analyze the results from the global epistasis model.

First, we will examine how the model looks on all the actual variants used to fit the model.
We use [AbstractEpistasis.phenotypes_df](https://jbloomlab.github.io/dms_variants/dms_variants.globalepistasis.html#dms_variants.globalepistasis.AbstractEpistasis.phenotypes_df) to get a data frame of all the variants used to fit each global epistasis model along with their functional scores and the latent and observed phenotypes predicted by each model.


```python
# NBVAL_IGNORE_OUTPUT

variants_df = pd.concat(
        [model.phenotypes_df
         .assign(library=lib,
                 likelihoodtype=likelihoodtype,
                 )
         for (epistasistype, likelihoodtype, lib), model in models.items()
         if (epistasistype == 'global epistasis')],
        ignore_index=True, sort=False)

#predictionsfile = os.path.join(config['global_epistasis_expr_dir'], 'globalepistasis_expression_predictions.csv')
#variants_df.to_csv(predictionsfile, index=False)
#print(f"Writing predictions to {predictionsfile}")

variants_df.head().round(2)
```




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>aa_substitutions</th>
      <th>func_score</th>
      <th>func_score_var</th>
      <th>latent_phenotype</th>
      <th>observed_phenotype</th>
      <th>library</th>
      <th>likelihoodtype</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>N13S L60P K94N S147T C150Y</td>
      <td>-3.01</td>
      <td>0.04</td>
      <td>-0.46</td>
      <td>-2.89</td>
      <td>lib1</td>
      <td>Gaussian</td>
    </tr>
    <tr>
      <th>1</th>
      <td>A22C R127G E141D L188V</td>
      <td>-2.54</td>
      <td>0.03</td>
      <td>-0.19</td>
      <td>-2.31</td>
      <td>lib1</td>
      <td>Gaussian</td>
    </tr>
    <tr>
      <th>2</th>
      <td>N13F</td>
      <td>-1.53</td>
      <td>0.01</td>
      <td>-0.15</td>
      <td>-1.91</td>
      <td>lib1</td>
      <td>Gaussian</td>
    </tr>
    <tr>
      <th>3</th>
      <td>C6K T15W K94Y V103W</td>
      <td>-4.25</td>
      <td>0.03</td>
      <td>-3.39</td>
      <td>-3.15</td>
      <td>lib1</td>
      <td>Gaussian</td>
    </tr>
    <tr>
      <th>4</th>
      <td>V71K P149L N157T</td>
      <td>-2.73</td>
      <td>0.02</td>
      <td>-0.30</td>
      <td>-2.84</td>
      <td>lib1</td>
      <td>Gaussian</td>
    </tr>
  </tbody>
</table>
</div>



Below we plot the relationships among the latent phenotype from the model, the observed phenotype from the model, and the measured functional score for all variants used to fit the model:


```python
for x, y in itertools.combinations(['latent_phenotype',
                                    'observed_phenotype',
                                    'func_score'],
                                   2):
    p = (
        ggplot(variants_df, aes(x, y)) +
        geom_point(alpha=0.05, size=0.5) +
        facet_grid('library ~ likelihoodtype', scales='free_y') +
        theme(figure_size=(2 * variants_df['likelihoodtype'].nunique(),
                           2 * variants_df['library'].nunique()),
              )
        )
#         plotfile = os.path.join(config['figs_dir'], f'{y}-v-{x}_by_{epistasistype}.pdf')
#         print(f"Saving to {plotfile}")
#         p.save(plotfile)
    _ = p.draw()
```


![png](global_epistasis_expression_files/global_epistasis_expression_24_0.png)



![png](global_epistasis_expression_files/global_epistasis_expression_24_1.png)



![png](global_epistasis_expression_files/global_epistasis_expression_24_2.png)


Shape seems pretty consistent between the two likelihood models. There are a fair number of points in the lower-right of the plots showing measured phenotype ("func_score") versus latent or predicted/observed phenotype, indicative of "false negatives" (that is, the model thinks they should be ~high expression, but experimentally, they were non-expressing). This is consistent with what I saw in the previous R script -- a handful of WT variants that had low expression, even with high sequencing counts (so, not noise). Some of these had high variant call support in PacBio sequencing, so they are likely veritable "WT" sequences -- I am assuming that they accrued mutations outside of the PacBio sequencing region (e.g. the 5' Gibson junction or elsewhere) and are therefore truly nonexpressing, though not by virtue of their scFv genotype. It is therefore actually a *good* sign that the model is finding these points, and saying they actually do have high latent phenotypes despite their poor experimental expression.

To get a better view into the "shape" of global epistasis, let's re-make the plots above but only showing single mutant barcodes.


```python
mask = (variants_df['aa_substitutions'].str.len() < 6) & (variants_df['aa_substitutions'].str.len() > 0)
single_variants_df = variants_df.loc[mask]
single_variants_df.head()
```




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>aa_substitutions</th>
      <th>func_score</th>
      <th>func_score_var</th>
      <th>latent_phenotype</th>
      <th>observed_phenotype</th>
      <th>library</th>
      <th>likelihoodtype</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>2</th>
      <td>N13F</td>
      <td>-1.53</td>
      <td>0.01</td>
      <td>-0.145062</td>
      <td>-1.914094</td>
      <td>lib1</td>
      <td>Gaussian</td>
    </tr>
    <tr>
      <th>6</th>
      <td>S184H</td>
      <td>-4.66</td>
      <td>0.01</td>
      <td>-0.028577</td>
      <td>-0.398932</td>
      <td>lib1</td>
      <td>Gaussian</td>
    </tr>
    <tr>
      <th>14</th>
      <td>P7S</td>
      <td>-0.07</td>
      <td>0.03</td>
      <td>-0.015319</td>
      <td>-0.184636</td>
      <td>lib1</td>
      <td>Gaussian</td>
    </tr>
    <tr>
      <th>27</th>
      <td>P149Q</td>
      <td>0.04</td>
      <td>0.02</td>
      <td>-0.021325</td>
      <td>-0.282752</td>
      <td>lib1</td>
      <td>Gaussian</td>
    </tr>
    <tr>
      <th>30</th>
      <td>D90Y</td>
      <td>-0.88</td>
      <td>0.05</td>
      <td>-0.089529</td>
      <td>-1.275370</td>
      <td>lib1</td>
      <td>Gaussian</td>
    </tr>
  </tbody>
</table>
</div>




```python
for x, y in itertools.combinations(['latent_phenotype',
                                    'observed_phenotype',
                                    'func_score'],
                                   2):
    p = (
        ggplot(single_variants_df, aes(x, y)) +
        geom_point(alpha=0.05, size=0.5) +
        facet_grid('library ~ likelihoodtype', scales='free_y') +
        theme(figure_size=(2 * variants_df['likelihoodtype'].nunique(),
                           2 * variants_df['library'].nunique()),
              )
        )
#         plotfile = os.path.join(config['figs_dir'], f'{y}-v-{x}_by_{epistasistype}.pdf')
#         print(f"Saving to {plotfile}")
#         p.save(plotfile)
    _ = p.draw()
```


![png](global_epistasis_expression_files/global_epistasis_expression_27_0.png)



![png](global_epistasis_expression_files/global_epistasis_expression_27_1.png)



![png](global_epistasis_expression_files/global_epistasis_expression_27_2.png)



```python
#add predicted and latent phenotypes to table by barcode with additional columns used for interpretation
dt = pd.read_csv(config['expression_sortseq_file'])
dt[['aa_substitutions']] = dt[['aa_substitutions']].fillna(value='')

dt = models.get(('global epistasis', 'Gaussian','lib1')).add_phenotypes_to_df(df=dt, substitutions_col='aa_substitutions',latent_phenotype_col='latent_phenotype_Gaussian_1',observed_phenotype_col='predicted_phenotype_Gaussian_1',unknown_as_nan=True)
dt = models.get(('global epistasis', 'Cauchy', 'lib1')).add_phenotypes_to_df(df=dt, substitutions_col='aa_substitutions',latent_phenotype_col='latent_phenotype_Cauchy_1',observed_phenotype_col='predicted_phenotype_Cauchy_1',unknown_as_nan=True)
dt = models.get(('global epistasis', 'Gaussian', 'lib2')).add_phenotypes_to_df(df=dt, substitutions_col='aa_substitutions',latent_phenotype_col='latent_phenotype_Gaussian_2',observed_phenotype_col='predicted_phenotype_Gaussian_2',unknown_as_nan=True)
dt = models.get(('global epistasis', 'Cauchy', 'lib2')).add_phenotypes_to_df(df=dt, substitutions_col='aa_substitutions',latent_phenotype_col='latent_phenotype_Cauchy_2',observed_phenotype_col='predicted_phenotype_Cauchy_2',unknown_as_nan=True)

dt.to_csv(config['global_epistasis_expr_file'], index=False)
print(f"Writing predictions to {config['global_epistasis_expr_file']}")
```

    Writing predictions to results/global_epistasis_expression/global_epistasis_expression_predictions.csv


## Repeat fits for pooled library measurements

Repeat the fits for all barcodes pooled together. There is slight variation in the average mean fluorescence ascribed to wildtype genotypes in each library. 


```python
df_joint = pd.read_csv(config['expression_sortseq_file'])
df_joint.rename(columns={'delta_ML_meanF':'func_score','var_ML_meanF':'func_score_var'},inplace=True)
func_scores_joint = df_joint[pd.notnull(df_joint['func_score'])]
func_scores_joint.fillna('',inplace=True)
func_scores_joint.head()
```




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>library</th>
      <th>target</th>
      <th>barcode</th>
      <th>variant_call_support</th>
      <th>total_count</th>
      <th>ML_meanF</th>
      <th>func_score</th>
      <th>func_score_var</th>
      <th>variant_class</th>
      <th>aa_substitutions</th>
      <th>n_aa_substitutions</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>1</th>
      <td>lib1</td>
      <td>SARS-CoV-2</td>
      <td>AAAAAAAAACTTAAAT</td>
      <td>2</td>
      <td>64.71</td>
      <td>7.45</td>
      <td>-3.01</td>
      <td>0.04</td>
      <td>&gt;1 nonsynonymous</td>
      <td>N13S L60P K94N S147T C150Y</td>
      <td>5</td>
    </tr>
    <tr>
      <th>2</th>
      <td>lib1</td>
      <td>SARS-CoV-2</td>
      <td>AAAAAAAAACTTCAAT</td>
      <td>5</td>
      <td>117.96</td>
      <td>7.92</td>
      <td>-2.54</td>
      <td>0.03</td>
      <td>&gt;1 nonsynonymous</td>
      <td>A22C R127G E141D L188V</td>
      <td>4</td>
    </tr>
    <tr>
      <th>3</th>
      <td>lib1</td>
      <td>SARS-CoV-2</td>
      <td>AAAAAAAACAAGCAGA</td>
      <td>6</td>
      <td>244.34</td>
      <td>8.93</td>
      <td>-1.53</td>
      <td>0.01</td>
      <td>1 nonsynonymous</td>
      <td>N13F</td>
      <td>1</td>
    </tr>
    <tr>
      <th>4</th>
      <td>lib1</td>
      <td>SARS-CoV-2</td>
      <td>AAAAAAAACAATATAA</td>
      <td>1</td>
      <td>95.35</td>
      <td>6.21</td>
      <td>-4.25</td>
      <td>0.03</td>
      <td>&gt;1 nonsynonymous</td>
      <td>C6K T15W K94Y V103W</td>
      <td>4</td>
    </tr>
    <tr>
      <th>5</th>
      <td>lib1</td>
      <td>SARS-CoV-2</td>
      <td>AAAAAAAACAGGTTGC</td>
      <td>4</td>
      <td>212.43</td>
      <td>7.73</td>
      <td>-2.73</td>
      <td>0.02</td>
      <td>&gt;1 nonsynonymous</td>
      <td>V71K P149L N157T</td>
      <td>3</td>
    </tr>
  </tbody>
</table>
</div>




```python
# NBVAL_IGNORE_OUTPUT

models_joint = {}  # store models, keyed by `(epistasistype, likelihoodtype)`

for (target), scores in func_scores_joint.groupby(['target']):
   
    bmap = dms_variants.binarymap.BinaryMap(scores)
    
    for epistasistype, likelihoodtype, Model in [
            ('global epistasis', 'Gaussian', dms_variants.globalepistasis.MonotonicSplineEpistasisGaussianLikelihood),
            ('no epistasis', 'Gaussian', dms_variants.globalepistasis.NoEpistasisGaussianLikelihood),
            ('global epistasis', 'Cauchy', dms_variants.globalepistasis.MonotonicSplineEpistasisCauchyLikelihood),
            ('no epistasis', 'Cauchy', dms_variants.globalepistasis.NoEpistasisCauchyLikelihood),
            ]:
        print(f"Fitting {epistasistype} with {likelihoodtype} likelihood model...", end=' ')
    
        start = time.time()
        model = Model(bmap)
        model.fit()  # do NOT change ftol in normal use, this is just for test
        print(f"fitting took {time.time() - start:.1f} sec.")
        models_joint[(epistasistype, likelihoodtype)] = model


```

    Fitting global epistasis with Gaussian likelihood model... fitting took 276.0 sec.
    Fitting no epistasis with Gaussian likelihood model... fitting took 1.0 sec.
    Fitting global epistasis with Cauchy likelihood model... fitting took 677.8 sec.
    Fitting no epistasis with Cauchy likelihood model... fitting took 8.4 sec.



```python
# NBVAL_IGNORE_OUTPUT

variants_df_joint = pd.concat(
        [model.phenotypes_df
         .assign(likelihoodtype=likelihoodtype)
         for (epistasistype, likelihoodtype), model in models_joint.items()
         if (epistasistype == 'global epistasis')],
        ignore_index=True, sort=False)

variants_df_joint.head().round(2)
```




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>aa_substitutions</th>
      <th>func_score</th>
      <th>func_score_var</th>
      <th>latent_phenotype</th>
      <th>observed_phenotype</th>
      <th>likelihoodtype</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>N13S L60P K94N S147T C150Y</td>
      <td>-3.01</td>
      <td>0.04</td>
      <td>-0.76</td>
      <td>-2.84</td>
      <td>Gaussian</td>
    </tr>
    <tr>
      <th>1</th>
      <td>A22C R127G E141D L188V</td>
      <td>-2.54</td>
      <td>0.03</td>
      <td>-0.30</td>
      <td>-2.17</td>
      <td>Gaussian</td>
    </tr>
    <tr>
      <th>2</th>
      <td>N13F</td>
      <td>-1.53</td>
      <td>0.01</td>
      <td>-0.26</td>
      <td>-1.95</td>
      <td>Gaussian</td>
    </tr>
    <tr>
      <th>3</th>
      <td>C6K T15W K94Y V103W</td>
      <td>-4.25</td>
      <td>0.03</td>
      <td>-5.53</td>
      <td>-3.30</td>
      <td>Gaussian</td>
    </tr>
    <tr>
      <th>4</th>
      <td>V71K P149L N157T</td>
      <td>-2.73</td>
      <td>0.02</td>
      <td>-0.56</td>
      <td>-2.82</td>
      <td>Gaussian</td>
    </tr>
  </tbody>
</table>
</div>




```python
for x, y in itertools.combinations(['latent_phenotype',
                                    'observed_phenotype',
                                    'func_score'],
                                   2):
    p = (
        ggplot(variants_df_joint, aes(x, y)) +
        geom_point(alpha=0.05, size=0.5))
#        facet_grid('likelihoodtype', scales='free_y') +
#        theme(figure_size=(2 * variants_df_novar['likelihoodtype'].nunique()),
#              )
#        )
#         plotfile = os.path.join(config['figs_dir'], f'{y}-v-{x}_by_{epistasistype}.pdf')
#         print(f"Saving to {plotfile}")
#         p.save(plotfile)
    _ = p.draw()
    

```


![png](global_epistasis_expression_files/global_epistasis_expression_33_0.png)



![png](global_epistasis_expression_files/global_epistasis_expression_33_1.png)



![png](global_epistasis_expression_files/global_epistasis_expression_33_2.png)




## Output epistasis model parameters


```python
#lib1 models
models.get(('global epistasis', 'Gaussian', 'lib1')).single_mut_effects(phenotype='latent',standardize_range=False).to_csv('results/global_epistasis_expression/Gaussian-latent-effects_expression_1.csv',index=False)
models.get(('global epistasis', 'Gaussian', 'lib1')).single_mut_effects(phenotype='observed',standardize_range=False).to_csv('results/global_epistasis_expression/Gaussian-predicted-effects_expression_1.csv',index=False)
models.get(('global epistasis', 'Cauchy', 'lib1')).single_mut_effects(phenotype='latent',standardize_range=False).to_csv('results/global_epistasis_expression/Cauchy-latent-effects_expression_1.csv',index=False)
models.get(('global epistasis', 'Cauchy', 'lib1')).single_mut_effects(phenotype='observed',standardize_range=False).to_csv('results/global_epistasis_expression/Cauchy-predicted-effects_expression_1.csv',index=False)
models.get(('no epistasis', 'Gaussian', 'lib1')).single_mut_effects(phenotype='observed',standardize_range=False).to_csv('results/global_epistasis_expression/nonepistatic-Gaussian-predicted-effects_expression_1.csv',index=False)
models.get(('no epistasis', 'Cauchy', 'lib1')).single_mut_effects(phenotype='observed',standardize_range=False).to_csv('results/global_epistasis_expression/nonepistatic-Cauchy-predicted-effects_expression_1.csv',index=False)
#lib2 models
models.get(('global epistasis', 'Gaussian', 'lib2')).single_mut_effects(phenotype='latent',standardize_range=False).to_csv('results/global_epistasis_expression/Gaussian-latent-effects_expression_2.csv',index=False)
models.get(('global epistasis', 'Gaussian', 'lib2')).single_mut_effects(phenotype='observed',standardize_range=False).to_csv('results/global_epistasis_expression/Gaussian-predicted-effects_expression_2.csv',index=False)
models.get(('global epistasis', 'Cauchy', 'lib2')).single_mut_effects(phenotype='latent',standardize_range=False).to_csv('results/global_epistasis_expression/Cauchy-latent-effects_expression_2.csv',index=False)
models.get(('global epistasis', 'Cauchy', 'lib2')).single_mut_effects(phenotype='observed',standardize_range=False).to_csv('results/global_epistasis_expression/Cauchy-predicted-effects_expression_2.csv',index=False)
models.get(('no epistasis', 'Gaussian', 'lib2')).single_mut_effects(phenotype='observed',standardize_range=False).to_csv('results/global_epistasis_expression/nonepistatic-Gaussian-predicted-effects_expression_2.csv',index=False)
models.get(('no epistasis', 'Cauchy', 'lib2')).single_mut_effects(phenotype='observed',standardize_range=False).to_csv('results/global_epistasis_expression/nonepistatic-Cauchy-predicted-effects_expression_2.csv',index=False)
#joint models
models_joint.get(('global epistasis', 'Gaussian')).single_mut_effects(phenotype='latent',standardize_range=False).to_csv('results/global_epistasis_expression/Gaussian-latent-effects_expression_joint.csv',index=False)
models_joint.get(('global epistasis', 'Gaussian')).single_mut_effects(phenotype='observed',standardize_range=False).to_csv('results/global_epistasis_expression/Gaussian-predicted-effects_expression_joint.csv',index=False)
models_joint.get(('global epistasis', 'Cauchy')).single_mut_effects(phenotype='latent',standardize_range=False).to_csv('results/global_epistasis_expression/Cauchy-latent-effects_expression_joint.csv',index=False)
models_joint.get(('global epistasis', 'Cauchy')).single_mut_effects(phenotype='observed',standardize_range=False).to_csv('results/global_epistasis_expression/Cauchy-predicted-effects_expression_joint.csv',index=False)
models_joint.get(('no epistasis', 'Gaussian')).single_mut_effects(phenotype='observed',standardize_range=False).to_csv('results/global_epistasis_expression/nonepistatic-Gaussian-predicted-effects_expression_joint.csv',index=False)
models_joint.get(('no epistasis', 'Cauchy')).single_mut_effects(phenotype='observed',standardize_range=False).to_csv('results/global_epistasis_expression/nonepistatic-Cauchy-predicted-effects_expression_joint.csv',index=False)



```


```python
#! jupyter nbconvert --to markdown global_epistasis_expression.ipynb --output-dir ./results/summary/ --output global_epistasis_expression.md
```


```python

```
