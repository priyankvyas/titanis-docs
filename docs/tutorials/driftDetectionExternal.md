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

The test set of the data is prepared for validation using a similar method. The code is presented below:

{: .note } Note the usage of the readStream method, instead of the read method in the previous code snippet. The reasoning for this will be provided shortly.

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

### Learner
{: .fs-5}

The task chosen for this experiment is a binary classification task. Hence, we have decided to use the Logistic Regression model from the spark.ml library.

### Drift Detector
{: .fs-5}

The drift detector used for this experiment is the ADWIN Change drift detector from MOA. The ADWIN Change drift detector is native-built in Java for the MOA machine learning library. However, Titanis enables the import of learners and drift detectors from MOA, using wrappers built in Titanis. For this experiment, we will use ADWIN Change with its default parameters. However, depending on the use-case, if a different drift detector from MOA is required, the drift detector can be chosen by importing the class, initializing the instance, and setting it as the MOA detector using the setMOADetectorObject method. To import the MOA drift detectors into the application, ensure that the MOA JAR file is included as a dependency, along with the Titanis JAR file. The motivation behind choosing a MOA drift detector for this experiment is to demonstrate how Titanis enables MOA drift detectors and other algorithms to be used from an external Spark application with ease. The initialization and usage is shown below.

```scala
import moa.classifiers.core.driftdetection.ADWINChangeDetector
import org.apache.spark.titanis.driftDetectors.{DriftDetector, MOADriftDetector}

{...}

/* Initialize the MOADriftDetector from Titanis. The default MOA drift detector used is CusumDM, but the drift
    detector can be changed using the MOADetectorObject param. In this instance, the ADWINChangeDetector is used
    instead. */
val driftDetector: MOADriftDetector = new MOADriftDetector()
driftDetector.init(resultSchema)
  .set(driftDetector.labelCol, "correctPredictions")
  .set(driftDetector.MOADetectorObject, new ADWINChangeDetector())

// Add the classification rate column to the predictions and set it as the target attribute for the drift detector.
val driftDetectorInput = results.selectExpr("*", "CASE WHEN (label = prediction) THEN 1 ELSE 0 END AS " +
  "correctPredictions")

// Initialize the drift detector query with the target attribute as the classification rate
driftDetector.detect(driftDetectorInput)
```

After initializing the MOADriftDetector wrapper from Titanis, we set the label column to "correctPredictions". For this experiment, the column we would like to observe for drift is the column that records boolean values for the true label matching the predicted label. Hence, we set the drift detector label column to this "correctPredictions" column. For your own experiment, if you wish to observe drift in another column, simply set the labelCol attribute of the drift detector to the desired column. Another property of the MOADriftDetector that we set is the MOADetectorObject property. We have set this to ADWINChangeDetector, but if you choose a different detector for your experiment, simply set the MOADetectorObject to be a new instance of your chosen detector. After setting these properties, to run the drift detection query, you only need to call the detect method on the input DataFrame.

{: .note } An important consideration to make when using the drift detectors from Titanis, and MOA, by extension, is that the input DataFrame for the drift detector needs to be streaming DataFrame. This is due to the architecture of the Titanis framework being optimized and built for the Spark Structured Streaming API. Titanis is developed as a streaming machine learning and data analysis library, hence all the learners and detectors utilize Streaming Queries to read the streaming DataFrames. For this experiment, the output of the learner can be converted to a streaming DataFrame using the code presented below. 
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

    // Initialize the testing streaming query.
    preprocessedTest.writeStream
      .queryName("TestQuery")
      .foreachBatch((batchDF: DataFrame, batchID: Long) => {

        /* Use the trained logistic regression model to predict the target attribute for the test data and write it to
        the results directory */
        val predictions = lrModel.transform(batchDF)
        predictions.drop("features", "rawPrediction", "probability").write.mode(SaveMode.Append)
          .option("mapreduce.fileoutputcommitter.marksuccessfuljobs", "false")
          .option("header", "true")
          .csv("./data/prediction_results")
      })
      .start()

    // Formulate the schema for the results data
    val resultSchema = StructType(Array(
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
      StructField("label", DoubleType, true,
        new MetadataBuilder().putString("type", "numeric").putString("identifier", "label").build()),
      StructField("id", DoubleType, true,
        new MetadataBuilder().putString("type", "numeric").putString("identifier", "id").build()),
      StructField("weight", DoubleType, true,
        new MetadataBuilder().putString("type", "numeric").putString("identifier", "weight").build()),
      StructField("prediction", DoubleType, true,
        new MetadataBuilder().putString("type", "numeric").putString("identifier", "prediction").build())))

    // Read the results as a streaming DataFrame to allow for the drift detector to initiate the streaming query
    val results = spark.readStream
      .format("csv")
      .option("header", "true")
      .schema(resultSchema)
      .load("./data/prediction_results/")
```

{: .note } The process to convert the static DataFrame to a streaming DataFrame begins with reading the test data as a streaming DataFrame. This is achieved by using the readStream method. The readStream method observes the loaded directory and treats each file in the directory as a micro-batch file. Following the reading, the streaming DataFrame is now passed to the StreamPrepareForLearning class for preprocessing. This preprocessing comprises renaming of columns and assembling the vector of predictor attributes. After the preprocessing, the preprocessed stream is processed using the writeStream query which allows for batch-wise processing of the micro-batches of a stream. For each micro-batch of the stream, the logistic regression model is used to make predictions on the data instances. The data with the predictions added on as a column is then written to an output directory in CSV format. This concludes the testing or prediction stage of the learner. Now that the results have been output to a file, we create another streaming DataFrame using the readStream method on the results file. The schema of the results DataFrame is provided explicitly in the code for simplicity. This whole process ensures that the drift detectors and learners from Titanis get the input data as a streaming DataFrame. This experiment goes one step further to test the data in a streaming fashion as well. However, if that is not required, only the code that describes the schema of the results DataFrame and the readStream function call in the final step is required for the Titanis detectors.

### Results
{: .fs-5}