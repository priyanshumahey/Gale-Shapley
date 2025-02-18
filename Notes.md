# GAN Data Matching
## Requirements
To run the code from this worklearn, you will be needing the following libraries:
- pandas
- numpy
- matplotlib
- pytorch
- matching.algorithms
Pandas is for the data as we want to work with it in Pandas dataframes, numpy is for mathematical operations, matplotlib is for plotting, pytorch is for creating torch tensors to speed up thee code and matching.algorithms contains the gale Shapley algorithm we'd want to use.


We will be doing the following import at the start:
```py
import pandas as pd
import numpy as np
import matplotlib as mpl
import matplotlib.pyplot as plt
import torch
from matching.algorithms import galeshapley
```

## Datasets
We start off by importing the real dataset which was taken off Kaggle. This dataset is the `'kag_risk_factors_cervical_cancer.csv'` dataset. 

The other datasets are stored under a folder called `Syn data` and they are named after the algorthim that was used to generate them as well as the amount of rows of data they have. 
We import in the realdata and the five synthetic dataset.
```py
realdata = pd.read_csv('kag_risk_factors_cervical_cancer.csv')
ctgan = pd.read_csv('Syn data/ctgan_5000.csv')
gauscop = pd.read_csv('Syn data/gauscop_5000.csv')
synthpop = pd.read_csv('Syn data/synthpop_5000.csv')
tvae = pd.read_csv('Syn data/tvae_5000.csv')
syndata = pd.read_csv('synData9.csv')
```

Then, we notice that the last 4 synthetic datasets have 34 columns instead of 36 so we will actually create another real dataset with only 34 columns to work with them better.

``` py
realdata34 = pd.read_csv('kag_risk_factors_cervical_cancer.csv')
realdata34.drop('STDs: Time since first diagnosis', axis=1, inplace=True)
realdata34.drop('STDs: Time since last diagnosis', axis=1, inplace=True)
```
When we want to compare with any dataset that has 34 columns only, we'd use realdata34. 

## Data Imputations

In the dataset, we start off with multiple missing values labeled with `?`.

In the data, there will need to be two different kind of data impuations. First, we may have to replace the missing value with the median value. Second, we may have to replace it with the mode value.

For the columns with continuous values (1,2,3,etc or 1.1,1.2,1.3,etc), we replace the missing values with the median value. Then, for the values that are binary, we use a mode imputation.

For binary data, they are often either mostly 1s or 0s. When there is missing data, depending on the rest of the data, it is either highly likely to be 1 or 0. This means for binary missing data, we figure out what the mode of that column is and use that to replace the rest of the missing data.


``` py
med34 = ['Age', 'Number of sexual partners', 'First sexual intercourse',
        'Num of pregnancies', 'Smokes (years)','Smokes (packs/year)','Hormonal Contraceptives (years)','IUD (years)',
        'STDs (number)', 'STDs: Number of diagnosis']

med = ['Age', 'Number of sexual partners', 'First sexual intercourse',
        'Num of pregnancies', 'Smokes (years)','Smokes (packs/year)','Hormonal Contraceptives (years)','IUD (years)',
        'STDs (number)', 'STDs: Number of diagnosis',
        'STDs: Time since first diagnosis', 'STDs: Time since last diagnosis']

bin = ['Smokes', 'Hormonal Contraceptives', 'IUD', 'STDs', 'STDs:condylomatosis',
        'STDs:cervical condylomatosis', 'STDs:vaginal condylomatosis',
        'STDs:vulvo-perineal condylomatosis', 'STDs:syphilis',
        'STDs:pelvic inflammatory disease', 'STDs:genital herpes',
        'STDs:molluscum contagiosum', 'STDs:AIDS', 'STDs:HIV',
        'STDs:Hepatitis B', 'STDs:HPV','Dx:Cancer', 'Dx:CIN', 'Dx:HPV', 'Dx', 'Hinselmann', 'Schiller',
        'Citology', 'Biopsy']
```

## Helper Functions

For the imputation, we will create a impute function we have named `impute()`. This function takes in a pandas dataset. That dataset is which will be imputed into. The graph (by deafult is False), allows us to create an initial histogram for us to view the data.

``` py
def impute(data, graph=False):
    temp_data = data
    temp_data[temp_data == '?'] = np.nan

    for i in temp_data.columns:
        if(temp_data[i].dtype == 'O'):
            temp_data[i] = temp_data[i].astype('float')
    if len(data.columns) == 34:
        med = ['Age', 'Number of sexual partners', 'First sexual intercourse',
        'Num of pregnancies', 'Smokes (years)','Smokes (packs/year)','Hormonal Contraceptives (years)','IUD (years)',
        'STDs (number)', 'STDs: Number of diagnosis']
    else:
        med = ['Age', 'Number of sexual partners', 'First sexual intercourse',
        'Num of pregnancies', 'Smokes (years)','Smokes (packs/year)','Hormonal Contraceptives (years)','IUD (years)',
        'STDs (number)', 'STDs: Number of diagnosis',
        'STDs: Time since first diagnosis', 'STDs: Time since last diagnosis']
        
    for i in med:
        imp = float(temp_data[i].median())
        temp_data[i].fillna(imp, inplace=True)
        temp_data[i].replace(np.nan, imp)

    bin = ['Smokes', 'Hormonal Contraceptives', 'IUD', 'STDs', 'STDs:condylomatosis',
        'STDs:cervical condylomatosis', 'STDs:vaginal condylomatosis',
        'STDs:vulvo-perineal condylomatosis', 'STDs:syphilis',
        'STDs:pelvic inflammatory disease', 'STDs:genital herpes',
        'STDs:molluscum contagiosum', 'STDs:AIDS', 'STDs:HIV',
        'STDs:Hepatitis B', 'STDs:HPV','Dx:Cancer', 'Dx:CIN', 'Dx:HPV', 'Dx', 'Hinselmann', 'Schiller',
        'Citology', 'Biopsy']

    for i in bin:
        imp = float(temp_data[i].mode())
        temp_data[i].fillna(imp, inplace=True)
        temp_data[i].replace(np.nan, imp)

    if graph:
        temp_data.hist(figsize=(25,15))
    
    return temp_data
```
There is a wide range of data present. To make it more easy to work with, we have a `norm()` function which will use min-max normalization to normalize the entire dataset.

``` py
def norm(data1):
    """
    Min-Max normalizes the continuous data
    """
    new_dat = data1.copy()
    for col in new_dat.columns:
        if col in med:
            denom = new_dat[col].max() - new_dat[col].min()
            if denom != 0:
                new_dat[col] = (new_dat[col] - new_dat[col].min()) / denom
            else:
                new_dat[col] = 0
    return new_dat
```

We then use those functions to do imputations and normalizations for the dataset.

``` py
realdataA = norm(impute(realdata))
syndataA = norm(impute(syndata))
ctganA = norm(impute(ctgan))
gauscopA = norm(impute(gauscop))
synthpopA = norm(impute(synthpop))
tvaeA = norm(impute(tvae))
realdata34A = norm(impute(realdata34))
```

## Distance Computations

For distance, we will need to create a method to compute the distance. For us, we wanted to create some method of scoring how similar the distances are.

This specific distance is similar to L1 distance and it is simply choosen in this project due to the fact it is simple to implement and it is an effective method. We do not need a complex metric at the moment but this metric can easily be replaced as needed.

Some categories are more important than others and for that reason, we created a weight option. The weights multiplies into the distance and it is significant as it lets us choose which distances are more important. We can edit the weights and select specific columns that are more important.

``` py
def compute_distance(df1, df2):
    
    ar1 = df1.to_numpy()
    ar2 = df2.to_numpy()
    x1 = torch.Tensor(ar1)
    x2 = torch.Tensor(ar2)

    y1 = x1.unsqueeze(2).expand(x1.size()[0], x1.size()[1], x2.size()[0])
    y2 = x2.unsqueeze(2).expand(x2.size()[0], x2.size()[1], x1.size()[0]).permute(2,1,0)

    weights = torch.ones(x1.size()[1])

    Δy = torch.abs(y1-y2)

    result = torch.matmul(Δy.permute(0,2,1), weights).numpy()
    
    return result
```

With this, we create scores for all the datasets, including real vs real.

``` python
syndataB = compute_distance(realdataA, syndataA)
ctganB = compute_distance(realdata34A, ctganA)
gauscopB = compute_distance(realdata34A, gauscopA)
synthpopB = compute_distance(realdata34A, synthpopA)
tvaeB = compute_distance(realdata34A, tvaeA)
realvreal = compute_distance(realdataA, realdataA)
real34vreal34 = compute_distance(realdata34A, realdata34A)
```

We make the synthetic datasets smaller, to the size of the real dataset, because it is easier to work with and lets us graph squares.

``` Py
newSyn = []
newCt = []
newGaus = []
newSynth = []
newTvae = []
realT = []

for i in range(858):
    newSyn.append(syndataB[i][:858].tolist())
    newCt.append(ctganB[i][:858].tolist())
    newGaus.append(gauscopB[i][:858].tolist())
    newSynth.append(synthpopB[i][:858].tolist())
    newTvae.append(tvaeB[i][:858].tolist())
    realT.append(realvreal[i][:858].tolist())
```
## Heatmaps

We create heatmaps and add a colorbar. The important things to pay attention to is the range of the colorbar. The darker squares are more far apart and the more yellow regions are closer together. 

We should pay special attention to how different the values in real data are from each other. This tells us how far apart real data points can be.

``` Py
plt.imshow(real34vreal34[:,:858], cmap=mpl.cm.get_cmap('cividis_r'))
plt.colorbar()
```
``` Py
plt.imshow(realvreal[:,:858], cmap=mpl.cm.get_cmap('cividis_r'))
plt.colorbar()
```
``` Py
plt.imshow(syndataB[:,:858], cmap=mpl.cm.get_cmap('cividis_r'))
plt.colorbar()
```
``` Py
plt.imshow(gauscopB[:,:858], cmap=mpl.cm.get_cmap('cividis_r'))
plt.colorbar()
```
``` Py
plt.imshow(synthpopB[:,:858], cmap=mpl.cm.get_cmap('cividis_r'))
plt.colorbar()
```
``` Py
plt.imshow(tvaeB[:858,:858], cmap=mpl.cm.get_cmap('cividis_r'))
plt.colorbar()
```

## Ranking
For the ranking system, we will utilize the Gale Shapley algorithm to match and rank the datasets. We use the Gale Shapley algorithm to match one row for the real dataset to one match for the specific synthetic dataset. 

We use the Gale Shapley algorithm due to the fact that it lets us match and choose the closest points. With this, we can see if the synthetic generation algorithm made a point of data that was too close and identifiable.

For the Gale Shapley algorithm, the fastest one we could find was from matching.algorithms. This library was faster than the one we created from scratch so we ultimately decided to go with this one.

The Gale Shapley algorthim takes in two different dictionaries which indicate the preferences for both datasets. So the first thing we need to is create the dictionary which is done by the `createDict()` function.

``` py
def createDict(A):
    s = torch.argsort(torch.Tensor(A),dim=1)
    d = {}
    for i in range(len(A)):
        d[i] = s[i,:].tolist()
    return d
```

Then, once we have created the dictionary, we will apply the galeshapley algorithm.

``` py
newSynM = newSyn

realDict = createDict(newSynM)
synDict = createDict(torch.tensor(newSynM).T)

matchNewSyn = galeshapley(realDict, synDict)
```

Now, we also create a graph of the patient vs the score of the match it was paired with.

``` py
y = []
for i in range(len(newSyn)):
    y.append(newSyn[i][matchNewSyn[i]])
x = np.array([i for i in range(858)])

plt.xlabel('Index of Real Patient') 
plt.ylabel('Score') 
plt.title("Real Data vs Score for Syndata")

plt.scatter(x, y)
plt.show()
```

## Graphing the Choices
For the matching algorithm, we would like to see if the final stable match has their first preference, or second, or third, or greater. For this, we create a new graph that color codes and visualizes the final results of the stable pair.

```Py
dataset_used = newCt

newSynM = dataset_used

realDict = createDict(newSynM)
synDict = createDict(torch.tensor(newSynM).T)
matchNewSynM = galeshapley(realDict, synDict)

dictUsed = realDict
matchUsed = matchNewSynM

y = []
for i in range(len(dataset_used)):
    y.append(dataset_used[i][matchNewSynM[i]])
x = np.array([i for i in range(858)])
plt.scatter(x, np.sort(y))
plt.show()

def choiceMap(rank, match):
    rankNew = rank
    matchNew = match
    d = {}
    for i in range(len(matchNew)):
        x = matchNew[i]
        x = rankNew[i].index(x)
        d[i] = x
    return d

map = choiceMap(dictUsed, matchUsed)
plt.scatter(map.keys(), map.values())
plt.show()

testPlot = map
X = list(testPlot.keys())
Y = list(testPlot.values())

color_lst = []
colors_set = {0:'green', 1:'blue', 2:'yellow', 3:'red', 4:'black'}

for i, j in enumerate(X):
    if Y[i] <= 3:
        color_lst.append(Y[i])
    if Y[i] > 3:
        color_lst.append(4)

    plt.scatter(X[i], Y[i], color = colors_set.get(color_lst[i], 'black'))
```