# The Political Opinion Classifier

This is a project by Yannick Gladow and Simon Wanner.
The goal is to correctly classify text data to political opinion based on data from reddit.

## The Idea
>It is said that if you know your enemies and know yourself, you will not be imperiled in a hundred battles; if you do not know your enemies but do know yourself, you will win one and lose one; if you do not know your enemies nor yourself, you will be imperiled in every single battle.

-- <cite>Sun Tzu</cite>

![structure](https://raw.githubusercontent.com/yannick-cw/poc_analyzer/master/sanders.png "sanders-twitter")

How cool would it be to just type in the twitter username `berniesanders` into a textfield and get this
```javascript
//actual results from poc
{
    "dem": 0.7148484082739541,
    "rep": 0.28515159172604543
}
```

![structure](https://raw.githubusercontent.com/yannick-cw/poc_analyzer/master/tea.png "tea twitter")

or this `tppatriots` (teaparty patriots)
```javascript
//actual results from poc
{
    "dem": 0.3883405523132041,
    "rep": 0.6116594476867955
}
```

Well thats what we thought.  
Find out if your new date is secretly a radical Trump supporter?  
Or your boss has the same policial views as you?

What if you could predict the political opinion of a person automatically by just looking at their posts on social media? 

This was the question, which lead to our idea, to classify text to different political groups.    

So we created a website where you can    
1. put in any text and classify it to democrats or republican  
2. link to a twitter account and classify this account to democrats or republicans 

We considered doing it for the German political spectrum, but  decided to try to differentiate between republicans and democrats, for the simple reason, that there are huge amounts of data available for US politics.  
As source for data we thought about using twitter, but decided to go for [reddit](http://reddit.com), because it is easier to find labeled training data there.
## The Data
Reddit is a huge datasource for labeled data, there are millions of posts to almost every existing topic. 
And these posts are upvoted or downvoted by other users of the site. 
Thereby labeled data, rated by acceptance, is created by the community.  
Although maybe not 100% accurate, we considered upvoted posts in subreddits of one political opinion to correspond to that political opinion.  
For our project we extracted posts from the following subreddits.
##### Democratic Subreddits
- [/democrats](http://reddit.com/r/democrats)
- [/liberal](http://reddit.com/r/liberal)
- [/SandersForPresident](http://reddit.com/r/SandersForPresident)
- [/hillaryclinton](http://reddit.com/r/hillaryclinton)
- [/obama](http://reddit.com/r/obama)

##### Republican Subreddits
- [/republican](http://reddit.com/r/republican)
- [/republicans](http://reddit.com/r/republicans)
- [/conservative](http://reddit.com/r/conservative)
- [/AskTrumpSupporters](http://reddit.com/r/AskTrumpSupporters)
- [/The_Donald](http://reddit.com/r/The_Donald)

From these two groups we gathered over 7GByte of raw json files containing roughly 4.000.000 distinct text documents, labeled with upvotes.

## The Software Architecture
We wanted to implement this project in a highly modular microservice fashion.  
As language we decided to go with [scala](http://www.scala-lang.org/), because for us it seemed to be the optimal fit for working with BigData and Machine Learning. 
Furthermore we tried to keep away from any blocking operations and comply to the [reactive manifesto.](http://www.reactivemanifesto.org/)

First we extracted the json data from reddit with the help of a python [script](https://github.com/peoplma/subredditarchive).  
After that the data is processed through multiple microservices, each communicating via a REST API.
  
#### [The Importer](https://github.com/yannick-cw/poc-importer)    
The importer reads in the json files from a directory and then processes them in a streaming fashion with [Akka Streams](http://doc.akka.io/docs/akka/2.4.7/scala/stream/index.html). 
 First each post from the parent post is is deserialized into an internal object representation, than it is sanitized, grouped into bulks and finally stored to an [elasticsearch](https://www.elastic.co/products/elasticsearch) database running in a docker container.

#### [The Cleaner](https://github.com/yannick-cw/poc_cleaner)  
`endpoint: /clean    `
```javascript
//input
{ "text" : "text to clean" }
//output
{ "cleanedText" : "after cleaning "}
```
The cleaner accepts http requests containing a text and gives back a text without stopwords and all words are stemmed with an implementation of the [Porter-Stemmer-Algorithm](https://de.wikipedia.org/wiki/Porter-Stemmer-Algorithmus).   
Furthermore relics from URL encoding are removed.
#### [The Analyzer](https://github.com/yannick-cw/poc_analyzer)
`endpoint: /classify`
```javascript
//input
{
	"algortihm" : "algorithm to use",
	"text" : "text to classify"
}
//output
//with probabilities for rep and dem
{
	"algorithm" : "algorithm used",
	"rep" : 0.5,
	"dem" : 0.5
}
```
The Analyzer is the heart of the whole project.   
First it reads in the labeled texts from the [elasticsearch](https://www.elastic.co/products/elasticsearch) database.  
After that multiple models are built with the classification algorithms that we chose.  
When the models are ready, it accepts http requests containing an algorithm name and a text and tries to classify the given text.
 Therefore the input text is first send to the Cleaner mircorservice and then classified by the selected algorithm.  

The seconds mode of operation is the validation phase. There each model is build and tested against a specified percentage of the input data. The input data is randomly distributed into test and train data.

The internal structure is based on the [Akka](http://akka.io/) Actor model.
#### [The Frontend](https://github.com/yannick-cw/poc_frontend)  
The Frontend is build with the [play frameworke](https://www.playframework.com/) and his main job is to send the user requests to the backend and display the results.

#### [The twitter linker](https://github.com/yannick-cw/poc_twitter_linker)
`endpoint /classifyUser`
```javascript
{ "username" : "the twitter username" }
```
The twitter linker provides a service to analyze twitter users's political opinion based on their tweets.The twitter linker gives the opportunity to the user to analyze twitter users's political opinion.  
In the frontend the username can be specified and then all recent posts of the twitter account are analyzed and the resulting political opinion is displayed.

Therefore the twitter linker has to crawl all recent posts of the input user. This is
implemented using the [twitter4s](https://github.com/DanielaSfregola/twitter4s) library. 

#### More Technologies used
To make it easy to deploy this everywhere, each service can be run in a [docker](https://www.docker.com/) container.  
The whole stack is running on an [AWS](https://aws.amazon.com) instance with 16GByte of RAM and 8 cores.  
For classification the machine learning library [weka](http://www.cs.waikato.ac.nz/ml/weka/) is used.

![structure](https://raw.githubusercontent.com/yannick-cw/poc_analyzer/master/pocStructure.png "Architecture")


## The Algorithms considered
#### The Featurevector
We tried different approaches to build a feature vector from the text to classify.   
One approach was to build a feature vector containing of:
- average spelling errors per word
- average text length
- average sentence length
- average distinct words
- average uppercase letters used
- average word length

From this approach we learned, that it does not work at all. All these features, combined or individually, where distributed very even between the political parties.  

So we decided to classify with the [bag of words approach](https://en.wikipedia.org/wiki/Bag-of-words_model).  
Furthermore we also tried to use n-grams but found no improvement in the classification results.

We considered the following algorithms, because they all seemed to create good results for text classification.

###### [naive bayes](https://en.wikipedia.org/wiki/Naive_Bayes_classifier)
The naive bayes classification is a standard procedure for text classification and so we chose to use this algorithm as the baseline for further explorations.

Naive bayes algorithms are based on the Bayes theorem and view each attribute from the featurevector as independent.  
The main idea is, that it calculates the probabilities of the input text for being in each class.  
Technical it looks like this:  
![structure](https://wikimedia.org/api/rest_v1/media/math/render/svg/9dd841d7c36e6d7449bea439ef99e8138810870d "class prop")  
Here the probability for input vector x<sub>1</sub>,...,x<sub>n</sub> belonging to class C<sub>k</sub> is calculated.  
Therefore the overall probability for C<sub>k</sub> is multiplied with the product of each probability for x<sub>i</sub> being in C<sub>k</sub>. In the end the Class with the highest probability is selected.



###### [bayes net](https://en.wikipedia.org/wiki/Bayesian_network)
The bayesian network builds on the bayes theorem and creates a probabilistic directed acyclic graph.
######  [mulinomial naive bayes](https://en.wikipedia.org/wiki/Naive_Bayes_classifier#Multinomial_naive_Bayes)
We use two different implementations of the multinomial naive bayes.  

1. the normal version [multinomial](http://weka.sourceforge.net/doc.dev/weka/classifiers/bayes/NaiveBayesMultinomial.html)  
2. and the version especially for text classification [multinomial text](http://weka.sourceforge.net/doc.dev/weka/classifiers/bayes/NaiveBayesMultinomialText.html)   


The difference of the used multinomial bayes classifiers to the naive bayes classifier is that it uses a multinomial distribution instead of the normal distribution.

######  [support vector machine](https://en.wikipedia.org/wiki/Support_vector_machine)
Support vector machines try to separate the data points in two classes. Therefore it tries to find the linear function keeping maximal distance to objects of each of the classes.
![structure](https://upload.wikimedia.org/wikipedia/commons/1/1b/Kernel_Machine.png "kernel trick")

With the kernel trick the multidimensional feature space is mapped to two dimensions and linear classification is possible.
######  [j48 tree](https://en.wikipedia.org/wiki/C4.5_algorithm)
j48 is an implementation of the C4.5 algorithm, a decision tree.  
It uses information entropy to split up the feature vector of the training data to create nodes in the tree. 
######  [k-nearest neighbors](https://en.wikipedia.org/wiki/K-nearest_neighbors_algorithm)
k-nearest neighbors has no model building phase. The whole algorithm is lazy evaluated. Meaning the calculation happens after the training phase, when actual data comes in.  
The new incoming data point is compared to its k nearest neighbors and then classified, usually according to the majority class of its neighbors.  

![structure](https://upload.wikimedia.org/wikipedia/commons/e/e7/KnnClassification.svg "knn")

--------------------------------------

All algorithms used are from the weka library, besides one version of naive bayes,  which we implemented ourselves.

#### Our naive bayes implementation
###### the model building
```scala
// Here classes are dem and rep with the input text documents
class BayesAlgorithm(classes: Class*) {
    //min times each word has to appear
    private val minWordAppearance: Int = 0

    //number of docs in class divided by number of all docs
    private val probabilityPerClass =
        classes.map(_class => _class.size.toDouble / classes.flatten.size.toDouble)

    //all distinct words from input texts
    private val vocabularySize = classes.flatten.flatten.distinct.size.toDouble

    //words in classes dem and rep
    private val wordsPerClass = classes.map(_.flatten.size)

    //function that maps the words to their count
    private val getPerWordCount: (Class) => Map[Word, Double] = _class => {
        _class.flatten
          .groupBy(identity)
          .mapValues(_.length.toDouble)
          .filter(_._2 >= minWordAppearance)
    }

    //creates maps for rep and dems each containing the count per word
    private val perClassWordAppearance = classes.map(getPerWordCount)
   
    //zip the map with the overall size of rep and dems
    private val zipped = wordsPerClass.zip(perClassWordAppearance)
}
```
###### the classification
```scala
override def classify(cleanedDoc: CleanedDoc): Seq[Double] = {
               
     //create list of words
     val inputText = cleanedDoc.cleanedText.split(" ")
     val zipped = wordsPerClass.zip(perClassWordAppearance)

     val classWiseProbabilities = zipped
                  .map { case (totalWordsClass, individualWordCountMap) => inputText
                    //replace each word with the appearance in class and add balance factor 1
                    .map { word => (individualWordCountMap.getOrElse(word, 0.0) + 1.0) / (totalWordsClass + vocabularySize) }
                  }

     classWiseProbabilities            
                  //multiply each word probability
                  .map(_.product)
                  //zip with general class probability
                  .zip(probabilityPerClass)
                  //mulitply with general class probability
                  .map { case (wordInClassProbability, generalClasProbability) => wordInClassProbability * generalClasProbability }
            }
```
## The Classification Results
For classification we decided to limit our dataset to posts with more than 20 upvotes. This provided us with the best results since the posts where obviously accepted by their community.  
This restricted the ~4.000.000 input documents to nearly 400.000 texts.  
We randomly distributed this data to 95% train and 5% test data.
If not stated differently our own sanitization was used.

|Algorithm   |% correct classified   |
|---|---|
|naive bayes **own implementation**  | 79,88%  |
|naive bayes multinomial text   | 79,67%  |
|SVM for text data with TF * IDF  | 78,4%  |
|SVW   | 77,28%  |
| J48 Tree  | 76,53%  |
| naive bayes multinomial  | 74,52%  |
| bayes net  | 74,5%  |
| naive bayes with TF * IDF | 70,28% |
| k-nearest neighbors  | 69,76%  |
| naive bayes  | 69,52%  |
| naive bayes, [lovins stemmer](http://snowball.tartarus.org/algorithms/lovins/stemmer.html), [rainbow stopwords](http://weka.sourceforge.net/doc.dev/weka/core/stopwords/Rainbow.html) | 68,46%
|  naive bayes, no sanitization | 52%  |

### Conclusion and algorithms used
The most important observation from these results was, that our own naive bayes implementation was the best and fastest by far. This has probably to do with the fact, that we could make use of parallel execution and fit the algorithm exactly to our needs.  

One more interesting result was, that the model building process for support vector machines, j48 tree and SVM for text data took more than 12 hours. So we decided that they are not practically for our approach, because we want faster startup times. 

In the end performance wise and by looking at the classification results we chose to use our naive bayes implementation, the bayes net, the k-nearest neighbors and the bayes multinomial for text data algorithms in our running project.  
Thereby we provide some diversity in results and the user decides which algorithm to trust.  


Altogether we are really happy with almost 80% correct classified data, the first tries ended in the 60% area and we tweaked a lot of parameters to get this 20% increase.

### Future Ideas
##### Add more Input possibilities
It would be very interesting to also include Facebook posts or texts from different social media sited. Furthermore twitter and facebook data could also be used to train the models.
##### Improve classification through user response
Another improvement would be to have a back channel from the user.  
The user could say if the classification seems to be correct and the model could be continually trained to create better results.

##### Run multiple algorithms in sequence to improve results
One way to push the correct classified percentage over 80% could be to include multiple algorithms in a single classification.
Unfortunately there was no time left to implement this feature.
##### Allow generic classes to classify to
In the long run we could imagine to allow generic classes, not only democrats or republicans. There would only be small changes necessary to achieve that, because most of the code is already build to allow any classes.




