---
title: Concept Drift Detection from an existing Spark application
layout: page
nav_order: 3
parent: Tutorials
---

# Concept Drift Detection from an external Spark application
{: .fs-9}

The following tutorial shows how to incorporate Titanis concept drift detection in an existing Spark application. The tutorial shows how to use a drift detector from MOA or Titanis in conjunction with a machine learning pipeline setup in a Spark application built with spark.ml or spark.mllib learners. It is recommended that Titanis is installed and setup using the steps given on the Setup page before starting the tutorial.

## Setup
{: .fs-7}

The experiment is performing a binary classification task using the spark.ml Logistic Regression model. The predictions made by the model are then fed to the Titanis drift detector to demonstrate how Titanis can be used in conjunction with other Spark libraries.

The setup for the experiment can be broken down into four components — the **data**, the **learner**, the **drift detector**, and the **results**.

### Data
{: .fs-5}

The data used for the experiment is the elecNormNew dataset, which is available online and also, provided with the repository as a sample dataset. The dataset is over 45,000 instances, but for the purposes of this experiment, only the first 10,000 instances are used. These instances are split into training and test set in the ratio 80:20. This data can be found in the data/batches/elecNormNew directory within the project directory. Alternatively, you can use any classification dataset to run this experiment.

For simplicity, the schema of the data is explicitly provided in the code, as shown below:
```scala
// Build the schema for the input data
val schema = StructType(Array(
  StructField("date", DoubleType, true,
    new MetadataBuilder().putString("type", "numeric").putString("identifier", "date").build()),
  StructField("day", DoubleType, true,
    new MetadataBuilder().putString("type", "numeric").putString("identifier", "day").build()),
  StructField("period", DoubleType, true,
    new MetadataBuilder().putString("type", "numeric").putString("identifier", "period").build()),
  StructField("nswprice", DoubleType, true,
    new MetadataBuilder().putString("type", "numeric").putString("identifier", "nswprice").build()),
  StructField("nswdemand", DoubleType, true,
    new MetadataBuilder().putString("type", "numeric").putString("identifier", "nswdemand").build()),
  StructField("vicprice", DoubleType, true,
    new MetadataBuilder().putString("type", "numeric").putString("identifier", "vicprice").build()),
  StructField("vicdemand", DoubleType, true,
    new MetadataBuilder().putString("type", "numeric").putString("identifier", "vicdemand").build()),
  StructField("transfer", DoubleType, true,
    new MetadataBuilder().putString("type", "numeric").putString("identifier", "transfer").build()),
  StructField("class", StringType, true,
    new MetadataBuilder().putLong("num_values", 2).putStringArray("values", Array("UP", "DOWN"))
      .putString("type", "nominal").putLong("UP", 0).putLong("DOWN", 1)
      .putString("identifier", "class").build())))
```
This explicit schema replicates the HEADER file schema for the dataset. There are eight numeric predictor attributes, and the target attribute is named "class", which has two distinct values &#8212; UP and DOWN. These categorical values are mapped to numeric values for processing. The metadata for all the attributes is also provided explicitly. If you are using your own data, ensure that the schema is built accurately either by explicitly providing it or by using a reader to parse the schema of the data.

The data is then read into a DataFrame using Spark. The DataFrame is then preprocessed and prepared for the learner and the drift detector. The learner requires the predictor attributes to be assembled in a vector. To ease the preprocessing step, Titanis has a stream preprocessor that assembles the vector and maps nominal attributes to numeric values, all using one class, StreamPrepareForLearning. We imported the StreamPrepareForLearning class from the Titanis JAR file and read the data using the Spark DataFrameReader, as shown below:
```scala
import org.apache.spark.titanis.preprocessing.StreamPrepareForLearning

{...}

// Load the training set from the data directory and specify the schema for the preprocessing and drift detector
val training = spark.read
  .format("csv")
  .option("header", "true")
  .schema(schema)
  .load("./data/training.csv")


/* Use the preprocessor to assemble a vector to be used for learning and rename it to features, and rename the
  target attribute to label. The Spark ML learner requires the predictor vector to have the attribute name, features
  and the target attribute to have the attribute name, label. Hence, we rename the variables before training. */
val preprocessedTraining = new StreamPrepareForLearning()
  .setTargetAttributeIdentifier("class")
  .setWeightAttributeIdentifier("weight")
  .setNominalFeaturesMap(training)
  .transform(training)
  .withColumnRenamed("X", "features")
  .withColumnRenamed("target", "label")
```

The test set of the data is prepared for validation using the same method. The code is presented below:
```scala
// Load the test data and preprocess it for testing.
val test = spark.readStream
  .format("csv")
  .option("header", "true")
  .schema(schema)
  .load("./data/test")
val preprocessedTest = new StreamPrepareForLearning()
  .setTargetAttributeIdentifier("class")
  .setWeightAttributeIdentifier("weight")
  .setNominalFeaturesMap(test)
  .transform(test)
  .withColumnRenamed("X", "features")
  .withColumnRenamed("target", "label")
```

Titanis' StreamPrepareForLearning class does multiple things. Firstly, it maps nominal attribute values to numeric values for the learners to be able to process them. Secondly, it adds the "id" and "weight" column to the DataFrame. The "id" column is an incremental counter that begins with 1 for each micro-batch of data that "arrives" to the application. The "weight" column is to specify the training weight of instances. In cases where the training weights are present in the data, the -w option for the Task will allow you to inform Titanis what the name of the weight attribute is. If there is no training weights provided, StreamPrepareForLearning adds a new "weight" column and assigns equal training weight to all the instances in the micro-batch. Lastly, StreamPrepareForLearning uses the transform function to assemble a vector of all predictor attributes and adds it to the DataFrame in a new column, "X". The name for the vector attribute and the target attribute is set to be the standard name for the features and label columns in Titanis i.e., "X" and "target". However, the spark.ml learners require the features and label attributes to have the attribute names, "features" and "label", so we also rename the columns to the required names as the final step of preprocessing.
