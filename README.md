# Predicting Review Usefulness with NLP

## Motivation

Yelp has gathered millions of reviews on various organizations, but not all reviews are equally useful. To measure usefulness, Yelp has asked the community to vote on the usefulness of each review. However, it often takes weeks or months for a good review to accumulate the votes it deserves. It would save much time spent searching if we can predict how useful a review would be as soon as it is written. 

The goal of this project is to build a predictive model based on the text alone, so there is no need to wait for people to vote and gather additional data. The target variable is the number of useful votes a review receives. Since most of the reviews have zero useful votes and the distribution is highly skewed to the right, I take the natural logarithm of the target variable so that the models are not biased towards reviews with lots of votes. To measure the predictive performance of a model, I use 5-fold cross-validation to generate an overall Root Mean Squared Error (RMSE) of the transformed target variable. 

## Table of Contents

1. [Text Preprocessing](https://github.com/andrewjsiu/Capstone_Project_NLP/blob/master/01%20Text_Preprocessing.ipynb)
2. [Predicting Review Usefulness with Word2Vec Features](https://github.com/andrewjsiu/Capstone_Project_NLP/blob/master/02%20Word2Vec.ipynb)
3. [Predicting Review Usefulness with Doc2Vec Features](https://github.com/andrewjsiu/Capstone_Project_NLP/blob/master/03%20Doc2Vec.ipynb)

## Text Preprocessing

[Yelp dataset](https://www.yelp.com/dataset_challenge) contains more than 4 million reviews written by 1 million users for 144 thousand businesses. The data are provided in .json format as separate files for businesses and reviews. The files are text files (UTF-8) with one json object corresponding to an individual record in each line.

To prepare the data in a more usable format, each business record is converted to a Python dictionary. Each business dictionary has a business_id as a unique identifier and an array of relevant categories that the business belongs to. I first keep all businesses that are in the “Health & Medical” category and then remove these health & medical related businesses that also belong to “Restaurants”, “Food”, or “Pets”.  To focus on English text, I also remove businesses in Germany and create a final set of business IDs for all 10,211 healthcare entities. 

Using this set of healthcare IDs, I create a new file that contains only the text from reviews for these healthcare organizations. Each review is written as a line in the new file, resulting in a total of 114,556 healthcare reviews. 

Certain words often appear right next to each other as a phrase or meaningful concept. For instance, birth control should not be considered as two separate words but as one phrase birth_control. I develop a phrase model by looping over all the words in the reviews and looking for words that tend to co-occur one after another, with a frequency that is much higher than what we would expect by random chance. The gensim library offers a Phrases class to detect common phrases from a stream of sentences. I run the phrase detection twice over the whole corpus to capture common phrases that are longer than two words such as american_dental_association. 

Before running the phrase modeling, I segment the reviews into sentences and use the spaCy library to lemmatize the text. After two passes of the phrase modeling, I remove all English stopwords and proper names and create a file of transformed text with one review per line. Now the data have two well-defined layers: documents for reviews and tokens for words and phrases. See my text-processing code [here](https://github.com/andrewjsiu/Capstone_Project_NLP/blob/master/01%20Text_Preprocessing.ipynb).

## Predicting Review Usefulness with Word2Vec Features

One way to vectorize the text is count the frequency of different words used and rely on the word-word co-occurrence matrix, leveraging the global statistical information. But it tends to perform poorly on word analogy, such as finding semantic or syntactic relationships that exist in pairs of words. Another method is based on local context windows, and the main idea is that the meaning of a word can be learned from its context. Consider the following sentence:

<img src="https://github.com/andrewjsiu/nlp-project/blob/master/images/sentence.png" height="110">

Imagine a sliding window that runs through the entire corpus with a fixed window size, say 7 words. The center word is surrounded by 3 context words before and 3 after it. These context words form the input layer as a bag of words, not considering the order of these words. Each word is encoded as a one-hot vector, so the dimension is the vocabulary size (V) with only one element set to one and the rest are zeros. This window slides through the corpus so it's called the continuous bag-of-words (CBOW) algorithm, which maximizes the conditional probability of actually observing the center word given the input context words in each sliding window. The skip-gram algorithm is the opposite, as it seeks to predict the context words given the center word.

<img src="https://github.com/andrewjsiu/nlp-project/blob/master/images/cbow.png" height="600">

At the core of the word2vec model is to train a neural network that produces a vector representation for each word in the corpus. Words that share common contexts will have similar word vectors.  For instance, the word vector for 'dentist' is most similar to word vectors for ‘pediatric_dentist’ and ‘orthodontist’.

To generate features for predictive modeling, one simple way is to average all vectors for words that appear in a review. For the healthcare reviews, there is a total of 6,382 terms in the word2vec vocabulary, with a dictionary mapping each word to a 100-dimensional semantic vector. Averaging all word vectors would yield 100 features for each review. Another version is to weight each word by the its term frequency and inverse document frequency (TF-IDF). Some have also suggested using the maximum vector plus the minimum vector in a review.

I find that linear regression performs poorly in predicting usefulness with an overall RMSE of more than 175. Ridge regression that controls for the problem of overfitting by penalizing large coefficients performs significantly better with an overall RMSE of 0.617. There is not much improvement if we use TF-IDF weighting on the average word vectors. Random Forest regressor further lowers the RMSE to 0.613, but the best performer is the XGBoost regressor achieving a RMSE of 0.603. See my code on predictive modeling with word2vec [here](https://github.com/andrewjsiu/Capstone_Project_NLP/blob/master/02%20Word2Vec.ipynb).

We can also play with the model to find which words are most useful or least useful. Since the document features are average word vectors, the predicted usefulness of a single word is that of a document that contains that one word vector. After estimating a regression model, it can be used to predict the number of useful votes that a hypothetical review that contains a single word vector will obtain. In ranking all words by their predicted usefulness, both XGBoost and Ridge regressors show that words that involve a dollar amount, 'price', 'co-pay' or 'cash' tend to be the most useful words, perhaps because they are informative and provide objective facts. Words like 'confidence', 'amazing' and 'excellent' are mere subjective feelings, so they are among the least useful of all words. Below is a table that shows the results for Ridge regression. 

No.|Ten Most Useful	Words | Ten Least Useful Words
---:|---------	| -----------
1 | parking	| exceptional
2 | price	| excellent
3 | few_minute | amazing
4 | become	| anyone
5 | co_pay	| great
6 | than	| expert
7 | amount	| outstanding
8 | side	| issue
9 | cash	| incredible
10 | pretty	| wonderful

## Predicting Review Usefulness with Doc2Vec Features

Another approach is to use the methods for learning word vectors to also learn document vectors. Th word vectors are trained to capture semantics by being asked to predict the next word in the sentence. The similar idea applies in the doc2vec model by asking the document vectors to also predict the next word given many contexts sampled from the document. This document token can be considered as another word. It captures what is missing in the current context or the topic of the document. This document vector is shared across all sliding windows generated from the same document. See the original paper by [Le and Mikolov](https://cs.stanford.edu/~quocle/paragraph_vector.pdf) (2014) for more detail. 

For every regression model, the predictive performance with Doc2Vec is better than that with features generated from Word2Vec. The linear regression with Doc2Vec achieves an overall RMSE of 0.597 with five folds of cross-validation. The best performer is still XGBoost regressor with a RMSE of 0.585. 

No.|Model | RMSE of 5-Fold CV | Description
---:|:--- | ---: | :---
1 | xgb_d2v	|  0.5850 | XGBoost Regressor with Doc2Vec features
2 | gbr_d2v	|  0.5853 | Gradient Boosting Regressor with Doc2Vec features
3 | rfr_d2v	|  0.5900 | Random Forest Regressor with Doc2Vec features
4 | ridge_d2v	|  0.5974 | Ridge Regression with Doc2Vec features
5 | lr_d2v	|  0.5974 | Linear Regression with Doc2Vec features
6 | xgb_w2v	|  0.6037 | XGBoost Regressor with average word vectors
7 | rfr_w2v	|  0.6130 | Random Forest Regressor with average word vectors
8 | ridge_w2v_tfidf	|  0.6175 | Ridge Regression with average word vectors weighted by TF-IDF
9 | ridge_w2v	|  0.6176 | Ridge Regression with average word vectors 
10 | lr_w2v_tfidf |175.3874 | Linear Regression with average word vectors weighted by TF-IDF
11 | lr_w2v |324.2977 | Linear Regression with average word vectors 

In practice, we may only have a small dataset, so I also check how predictive performance varies with the amount of training data. Unsurprisingly, RMSE falls as we use more training data from 30% to 70% of the entire corpus, but the decrease in error is less than 0.003 for all models. Thus, even if we only use 30% of the data, we can still achieve reasonably good out-of-sample prediction. 

To further improve the predictive performance, we can tune the several parameters of XGBoost regressor, such as the maximum depth of a tree which determines the complexity of the tree, subsample ratio of the training instance for growing trees, and the degree of regularization on weights. In the end, using the best values for these parameters found by a grid search cross validation the RMSE on the test set falls to 0.5821. See my code on predictive modeling with doc2vec [here](https://github.com/andrewjsiu/Capstone_Project_NLP/blob/master/03%20Doc2Vec.ipynb).

<img src="https://github.com/andrewjsiu/nlp-project/blob/master/images/feature_imp.png" height="600">

## Conclusion

This project provides a model that can predict the usefulness of a review based on the text alone, without having to wait for the community to vote on review usefulness. In this project, I process the text of Yelp reviews by lemmatizing the words, finding common phrases, removing English stopwords. I then train the word2vec model to learn word embeddings by making predictions based on local context windows and the doc2vec model that assigns additionally a vector for each document. The results show that the feature vectors obtained from the doc2vec model combined with the XGBoost estimator generated the highest predictive performance. 

There are several ways to further improve the analysis. First, instead of training word vectors based on narrow context windows, we can train [global vectors for word representation](https://nlp.stanford.edu/projects/glove/) (GloVe) by making use of the global word-to-word co-occurrence matrix. GloVe consists of a weighted least squares model that trains on such co-occurrence counts. Second, the results are likely to improve if we also include other features, such as the star rating, the text length and the number of reviews the reviewer has written. Lastly, I could try a newer and faster gradient boosting framework called the Light Gradient Boosting Machine (LightGBM), which finds the best split among all tree leaves instead of being constrained by the tree depth. 

