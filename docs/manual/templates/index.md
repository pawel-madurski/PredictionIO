---
layout: mllib
title: Engine Templates
---

# Engine Templates

PredictionIO currently offers two engine templates for **Apache Spark MLlib**:

* Collaborative Filtering Engine Template - with MLlib ALS
* Classification Engine Template - with MLlib Naive Bayes 

This tutorial shows you how to use the Collaborative Filtering Engine Template to build your own recommendation engine for production use.
The usage of other engine templates are very similar.

> What is an Engine Template?
> 
> It is a basic skeleton of an engine. You can customize it easily to fit your specific needs. 

PredictionIO offers the following features on top of Apache Spark  MLlib project:

* Deploy Spark MLlib model as a service on a production environment
* Support JSON query to retrieve prediction online
* Offer separation-of-concern software pattern based on the DASE architecture
* Support model update with new data

# Quick Start

## Install PredictionIO

First you need to [install PredictionIO 0.8.1 or above]({{site.baseurl}}/templates/install-for-mllib.html)

## Create a new Engine from an Engine Template

First, you start a new engine called *MyEngine* by cloning the MLlib Collaborative Filtering engine template: 

```
$ cp $PIO_HOME/templates/scala-parallel-recommendation MyEngine
$ cd MyEngine
```
where $PIO_HOME is the installation directory of PredictionIO.

By default, the engine reads training data from a text file located at data/sample_movielens_data.txt. Use the sample movie data from MLlib repo for now:

```
$ curl https://raw.githubusercontent.com/apache/spark/master/data/mllib/sample_movielens_data.txt --create-dirs -o data/sample_movielens_data.txt
```

## Deploy the Engine as a Service

To build *MyEngine* and deploy it as a service:

```
$ $PIO_HOME/bin/pio build
$ $PIO_HOME/bin/pio train
$ $PIO_HOME/bin/pio deploy
```

This will deploy an engine that binds to http://localhost:8000. You can visit that page in your web browser to check its status.

Now, You can try to retrieve predicted results.
To recommend 4 movies to user 1, you send this JSON { "user": 1, "num":4 } to the deployed engine and it will return a JSON of the recommended movies.

```
$ curl -H "Content-Type: application/json" -d '{ "user": 1, "num":4 }' http://localhost:8000/queries.json

{"productScores":[{"product":22,"score":4.072304374729956},{"product":62,"score":4.058482414005789},{"product":75,"score":4.046063009943821},{"product":68,"score":3.8153661512945325}]}
```

Your MyEngine is now running. Next, we are going to take a look at the engine architecture and explain how you can customize it completely.


# Introduction to DASE Architecture

PredictionIO's DASE architecture brings the separation-of-concerns design principle to predictive engine development.
DASE stands for the following components of an engine:

* **D**ata - includes Data Source and Data Preparator 
* **A**lgorithm(s)
* **S**erving
* **E**valuator

Let's look at the code and see how you can customize the engine.

> Note: Evaluator will not be covered in this tutorial.

## The Engine Design

As you can see from the above, *MyEngine* takes a JSON prediction query, e.g. { "user": 1, "num":4 } and return
the predicted result in JSON format. 

In MyEngine/src/main/scala/***Engine.scala***

`Query` case class defines the format of **query**, such as { "user": 1, "num":4 }:

```
case class Query(
  val user: Int,
  val num: Int
) extends Serializable
```

`PredictedResult` case class defines the format of **predicted result**, such as {"productScores":[{"product":22,"score":4.07},{"product":62,"score":4.05},{"product":75,"score":4.04},{"product":68,"score":3.81}]}: 

```
case class PredictedResult(
  val productScores: Array[ProductScore]
) extends Serializable

case class ProductScore(
  product: Int,
  score: Double
) extends Serializable
```

Finally, `RecommendationEngine` is the Engine Factory that defines the components this engine will use: 
Data Source, Data Preparator, ALgorithm(s) and Serving components.

```
object RecommendationEngine extends IEngineFactory {
  def apply() = {
    new Engine(
      classOf[DataSource],
      classOf[Preparator],
      Map("als" -> classOf[ALSAlgorithm]),
      classOf[Serving])
  }
  
```

### Spark MLlib

Spark's MLlib ALS algorithm takes training data of RDD type, i.e. `RDD[Rating]` and train a model, which is a `MatrixFactorizationModel` object.

PredictionIO's MLlib Collaborative Filtering engine template, which *MyEngine* bases on, integrates this algorithm under the DASE architecture. 
We will take a closer look at the DASE code below.
> [Check this out](https://spark.apache.org/docs/latest/mllib-collaborative-filtering.html) to learn more about MLlib's ALS collaborative filtering algorithm.


## Data

In the DASE architecture, data is prepared by 2 components sequentially: *Data Source* and *Data Preparator*.
*Data Source* and *Data Preparator* takes data from the data store and prepares `RDD[Rating]` for the ALS algorithm.

### Data Source

In MyEngine/src/main/scala/***DataSource.scala***

The `def readTraining` of class `DataSource` reads, and selects, data from a data store and it returns `TrainingData`.

```
class DataSource(val dsp: DataSourceParams)
  extends PDataSource[DataSourceParams, EmptyDataParams, TrainingData, Query, EmptyActualResult] {

  override
  def readTraining(sc: SparkContext): TrainingData = {
    val data = sc.textFile(dsp.filepath)
    val ratings: RDD[Rating] = data.map(_.split("::") match {
      case Array(user, item, rate) =>
        Rating(user.toInt, item.toInt, rate.toDouble)
    })
    new TrainingData(ratings)
  }
}
```

As seen, it reads from a text file with `sc.textFile` by default.
PredictionIO automatically loads the parameters of *datasource* specified in MyEngine/***engine.json***, including *filepath*, to `dsp`.

In ***engine.json***:

``` 
datasource: {
  "filepath": "./data/sample_movielens_data.txt"
}
```

In this sample text data file, values are delimited by double colons (::). The first column are user IDs. The second column are item IDs. The third column are ratings. 


The class definition of `TrainingData` is:

```
class TrainingData(
  val ratings: RDD[Rating]
) extends Serializable
```
and PredictionIO passes the returned `TrainingData` object to *Data Preparator*.

> HOW-TO:
>
> You may modify readTraining function to read from other datastores, such as MongoDB -  [link]


 
### Data Preparator

In MyEngine/src/main/scala/***Preparator.scala***

The `def prepare` of class `Preparator` takes `TrainingData`. It then conducts any necessary feature selection and data processing tasks.
At the end, it returns `PreparedData` which should contain the data *Algorithm* needs. For MLlib ALS, it is `RDD[Rating]`.

By default, `prepare` simply copies the unprocessed `TrainingData` data to `PreparedData`:

```
class Preparator
  extends PPreparator[EmptyPreparatorParams, TrainingData, PreparedData] {

  def prepare(sc: SparkContext, trainingData: TrainingData): PreparedData = {
    new PreparedData(ratings = trainingData.ratings)
  }
}

class PreparedData(
  val ratings: RDD[Rating]
) extends Serializable
```

PredictionIO passes the returned `PreparedData` object to Algorithm's `train` function.

> HOW-TO:
> 
> MLlib ALS limitation: user id, item id must be integer - convert [link]


## Algorithm

In MyEngine/src/main/scala/***ALSAlgorithm.scala***

The two functions of the algorithm class are `def train` and `def predict`.
`def train` is responsible for training a predictive model. PredictionIO will store this model and `def predict` is responsible for using this model to make prediction. 

### def train

`def train` is called when you run **pio train**.  This is where MLlib ALS algorithm, i.e. `ALS.train`, is used to train a predictive model.

```
  def train(data: PreparedData): PersistentMatrixFactorizationModel = {
    val m = ALS.train(data.ratings, ap.rank, ap.numIterations, ap.lambda)
    new PersistentMatrixFactorizationModel(
      rank = m.rank,
      userFeatures = m.userFeatures,
      productFeatures = m.productFeatures)
  }
```

In addition to `RDD[Rating]`, `ALS.train` takes 3 parameters: *rank*, *iterations* and *lambda*. 

PredictionIO automatically loads the parameters of *algorithms* specified in MyEngine/***engine.json*** to the constructor `ap` of class `ALSAlgorithmParams`:

```
case class ALSAlgorithmParams(
  val rank: Int,
  val numIterations: Int,
  val lambda: Double,
  val persistModel: Boolean) extends Params
```

And in ***engine.json***:

``` 
{
 algorithms: [
      {
        "name": "als",
        "params": {
          "rank": 10,
          "numIterations": 20,
          "lambda": 0.01,
          "persistModel": true
        }
      }
    ]
}
```

`ALS.train` then returns a `MatrixFactorizationModel` model, which contains RDD data. 
RDD is a distributed collection of items which *does not* persist. To store the model, `PersistentMatrixFactorizationModel` extends `MatrixFactorizationModel` and makes it persistable.

> The detailed implementation can be found at MyEngine/src/main/scala/***PersistentMatrixFactorizationModel.scala***

PredictionIO will automatically store the returned model, i.e. `PersistentMatrixFactorizationModel` in this case. 


### def predict

`def predict` is called when you send a JSON query to http://localhost:8000/queries.json. PredictionIO converts the query, such as  { "user": 1, "num":4 } to the `Query` class you defined previously.  

The predictive model `MatrixFactorizationModel` of MLlib ALS, which is now extended as `PersistentMatrixFactorizationModel`,
offers a function called `recommendProducts`. `recommendProducts` takes two parameters: user id (i.e. query.user) and the number of products to be returned (i.e. query.num).
It predicts the top *num* of products a user will like.

```
def predict(
    model: PersistentMatrixFactorizationModel,
    query: Query): PredictedResult = {
    // MLlib MatrixFactorizationModel return Array[Rating]
    val productScores = model.recommendProducts(query.user, query.num)
      .map (r => ProductScore(r.product, r.rating))
    new PredictedResult(productScores)
}
```
 
> You have defined the class `PredictedResult` earlier. 

PredictionIO passes the returned `PredictedResult` object to *Serving*.

## Serving

The `def serve` of class `Serving` processes predicted result. It is also responsible for combining multiple predicted results into one if you have more than one predictive model.
*Serving* then returns the final predicted result. PredictionIO will convert it to a JSON response automatically. 

In MyEngine/src/main/scala/***Serving.scala***

```
class Serving
  extends LServing[EmptyServingParams, Query, PredictedResult] {

  override
  def serve(query: Query, predictions: Seq[PredictedResult]): PredictedResult = {
    predictions.head
  }
}
```

When you send a JSON query to http://localhost:8000/queries.json, `PredictedResult` from all models
will be passed to `def serve` as a sequence, i.e. `Seq[PredictedResult]`.

> An engine can train multiple models if you specify more than one Algorithm component in `object RecommendationEngine` inside ***Engine.scala*.
>
> Since only one ALSAlgorithm is implemented by default, this Sequence contains one element.

In this case, `def serve` simply returns the predicted result of the first, and the only, algorithm, i.e. `predictions.head`.

> HOW-TO:
> 
> Recommend products that the targeted user has not seen before [link]
>
> Give higher priority to newer products
>
> Combining several predictive model to improve prediction accuracy

# Model Re-training

You can update the predictive model with new data by making the *train* and *deploy* commands again:

1.  Assuming you already have a running engine from the previous
    section, go to http://localhost:8000 to check its status. Take note of the
    **Instance ID** at the top.

2.  Run training and deploy again. There is no need to manually terminate the previous deploy instance.

    ```
    $ $PIO_HOME/bin/pio train
    $ $PIO_HOME/bin/pio deploy
    ```

3.  Refresh the page at http://localhost:8000, you should see the status page with a new **Instance ID** at the top.
    
    
For example, if you want to re-train the model every day, you may add this to your *crontab*:

```
0 0 * * *   $PIO_HOME/bin/pio train; $PIO_HOME/bin/pio deploy
```

Congratulations! You have just learned how to customize and build a production-ready engine. Have fun!