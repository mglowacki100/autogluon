# AutoMM for Text + Tabular - Quick Start
:label:`sec_automm_textprediction_multimodal`

In many applications, text data may be mixed with numeric/categorical data. 
AutoGluon's `MultiModalPredictor` can train a single neural network that jointly operates on multiple feature types, 
including text, categorical, and numerical columns. The general idea is to embed the text, categorical and numeric fields 
separately and fuse these features across modalities. This tutorial demonstrates such an application.


```{.python .input}
import numpy as np
import pandas as pd
import warnings
import os

warnings.filterwarnings('ignore')
np.random.seed(123)
```

```{.python .input}
!python3 -m pip install openpyxl
```

## Book Price Prediction Data

For demonstration, we use the book price prediction dataset from the [MachineHack Book Price Prediction Hackathon](https://www.machinehack.com/hackathons/predict_the_price_of_books/overview). Our goal is to predict a book's price given various features like its author, the abstract, the book's rating, etc.

```{.python .input}
!mkdir -p price_of_books
!wget https://automl-mm-bench.s3.amazonaws.com/machine_hack_competitions/predict_the_price_of_books/Data.zip -O price_of_books/Data.zip
!cd price_of_books && unzip -o Data.zip
!ls price_of_books/Participants_Data
```

```{.python .input}
train_df = pd.read_excel(os.path.join('price_of_books', 'Participants_Data', 'Data_Train.xlsx'), engine='openpyxl')
train_df.head()
```

We do some basic preprocessing to convert `Reviews` and `Ratings` in the data table to numeric values, and we transform prices to a log-scale.

```{.python .input}
def preprocess(df):
    df = df.copy(deep=True)
    df.loc[:, 'Reviews'] = pd.to_numeric(df['Reviews'].apply(lambda ele: ele[:-len(' out of 5 stars')]))
    df.loc[:, 'Ratings'] = pd.to_numeric(df['Ratings'].apply(lambda ele: ele.replace(',', '')[:-len(' customer reviews')]))
    df.loc[:, 'Price'] = np.log(df['Price'] + 1)
    return df
```

```{.python .input}
train_subsample_size = 1500  # subsample for faster demo, you can try setting to larger values
test_subsample_size = 5
train_df = preprocess(train_df)
train_data = train_df.iloc[100:].sample(train_subsample_size, random_state=123)
test_data = train_df.iloc[:100].sample(test_subsample_size, random_state=245)
train_data.head()
```

## Training

We can simply create a MultiModalPredictor and call `predictor.fit()` to train a model that operates on across all types of features. 
Internally, the neural network will be automatically generated based on the inferred data type of each feature column. 
To save time, we subsample the data and only train for three minutes.


```{.python .input}
from autogluon.multimodal import MultiModalPredictor
import uuid

time_limit = 3 * 60  # set to larger value in your applications
model_path = f"./tmp/{uuid.uuid4().hex}-automm_text_book_price_prediction"
predictor = MultiModalPredictor(label='Price', path=model_path)
predictor.fit(train_data, time_limit=time_limit)
```

## Prediction

We can easily obtain predictions and extract data embeddings using the MultiModalPredictor.


```{.python .input}
predictions = predictor.predict(test_data)
print('Predictions:')
print('------------')
print(np.exp(predictions) - 1)
print()
print('True Value:')
print('------------')
print(np.exp(test_data['Price']) - 1)

```

```{.python .input}
performance = predictor.evaluate(test_data)
print(performance)
```

```{.python .input}
embeddings = predictor.extract_embedding(test_data)
embeddings.shape
```

## What's happening inside?
:label:`sec_automm_textprediction_architecture`

Internally, we use different networks to encode the text columns, categorical columns, and numerical columns. The features generated by individual networks are aggregated by a late-fusion aggregator. The aggregator can output both the logits or score predictions. The architecture can be illustrated as follows:

![Multimodal Network with Late Fusion](https://autogluon-text-data.s3.amazonaws.com/figures/fuse-late.png)
:width:`600px`

Here, we use the pretrained NLP backbone to extract the text features and then use two other towers to extract the feature from categorical column and the numerical column.

In addition, to deal with multiple text fields, we separate these fields with the `[SEP]` token and alternate 0s and 1s as the segment IDs, which is shown as follows:

![Preprocessing](https://autogluon-text-data.s3.amazonaws.com/figures/preprocess.png)
:width:`600px`

## How does this compare with TabularPredictor?

Note that `TabularPredictor` can also handle data tables with text, numeric, and categorical columns, but it uses an ensemble of many types of models and may featurize text. `MultiModalPredictor` instead directly fuses multiple neural network models directly and handles 
raw text (which are also capable of handling additional numerical/categorical columns). We generally recommend `TabularPredictor` if your table contains mainly numeric/categorical columns and MultiModalPredictor if your table contains mainly text columns, 
but you may easily try both and we encourage this. In fact, `TabularPredictor.fit(..., hyperparameters='multimodal')` will train a MultiModalPredictor along with many other tabular models and ensemble them together. 
Refer to the tutorial ":ref:`sec_tabularprediction_text_multimodal`"  for more details.

## Other Examples

You may go to [AutoMM Examples](https://github.com/autogluon/autogluon/tree/master/examples/automm) to explore other examples about AutoMM.

## Customization
To learn how to customize AutoMM, please refer to :ref:`sec_automm_customization`.