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

{: .note} 
Note the usage of the readStream method, instead of the read method in the previous code snippet. The reasoning for this will be provided shortly.

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

{: .note} 
An important consideration to make when using the drift detectors from Titanis, and MOA, by extension, is that the input DataFrame for the drift detector needs to be streaming DataFrame. This is due to the architecture of the Titanis framework being optimized and built for the Spark Structured Streaming API. Titanis is developed as a streaming machine learning and data analysis library, hence all the learners and detectors utilize Streaming Queries to read the streaming DataFrames. For this experiment, the output of the learner can be converted to a streaming DataFrame using the code presented below.

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

{: .note} 
The process to convert the static DataFrame to a streaming DataFrame begins with reading the test data as a streaming DataFrame. This is achieved by using the readStream method. The readStream method observes the loaded directory and treats each file in the directory as a micro-batch file. Following the reading, the streaming DataFrame is now passed to the StreamPrepareForLearning class for preprocessing. This preprocessing comprises renaming of columns and assembling the vector of predictor attributes. After the preprocessing, the preprocessed stream is processed using the writeStream query which allows for batch-wise processing of the micro-batches of a stream. For each micro-batch of the stream, the logistic regression model is used to make predictions on the data instances. The data with the predictions added on as a column is then written to an output directory in CSV format. This concludes the testing or prediction stage of the learner. Now that the results have been output to a file, we create another streaming DataFrame using the readStream method on the results file. The schema of the results DataFrame is provided explicitly in the code for simplicity. This whole process ensures that the drift detectors and learners from Titanis get the input data as a streaming DataFrame. This experiment goes one step further to test the data in a streaming fashion as well. However, if that is not required, only the code that describes the schema of the results DataFrame and the readStream function call in the final step is required for the Titanis detectors.

### Results
{: .fs-5}

The results of this experiment are split into two parts. For the training and prediction part of the experiment, the learned model will be output to the file that saves the console output, and the predictions will be output to a CSV file in the data/prediction_results directory, as specified in the code for the write method. The prediction results will be presented as the data instance observed with a few additional attributes, namely "id", "weight", and "prediction". For the drift detection part of the experiment, the results will be presented in a CSV file with the input stream instances printed along with a few additional columns, namely “id”, “weight”, and “detectedChange.” The id column in both the result files is a column of longs which acts as a row count for the micro-batch instances. Similarly, the weight column present in results for both parts of the experiment is a column of doubles which is the weight given to each instance of the micro-batch. The prediction column observed in the results file for the prediction part is a numeric column with discrete values. The value 0.0 indicates that the predicted class for that instance is class 0, and the value 1.0 indicates that the predicted class for the instance is class 1. The detectedChange column is a boolean column, where a value of 0.0 indicates that no change or no concept drift was detected at that instance, and a value of 1.0 indicates that change or concept drift was detected at that instance. The instances at which concept drift is detected are output to the console as well. This format of the results should facilitate the visualization or further analysis that might be conducted after the prediction or drift detection stages are completed.

## Methodology
{: .fs-7}

1. Navigate to the project directory.
2. Open a Terminal in the project root. Alternatively, you can navigate to the project directory using a Terminal, if opened in a different location.
3. Build the project using the following command in the Terminal:
    ```shell
    sbt package
    ```
4. After the build is completed, a success message will be printed. Following this, you can use the following command to run the experiment as described in the Setup section above:  
   **Windows**
   ```shell
   spark-submit --class "org.apache.spark.spark_ml_titanis.classification" --master local[*] --jars ".\lib\moa.jar",".\lib\titanis_2.13-0.1.jar" ".\target\scala-2.13\spark-ml-titanis_2.13-0.1.0-SNAPSHOT.jar" 1>.\results.csv 2>.\log.log
   ```  
   **macOS/Linux**
   ```shell
   spark-submit --class "org.apache.spark.spark_ml_titanis.classification" --master local[*] --jars "./lib/moa.jar","./lib/titanis_2.13-0.1.jar" "./target/scala-2.13/spark-ml-titanis_2.13-0.1.0-SNAPSHOT.jar" 1>./results.csv 2>./log.log
   ```
   If you are using your own data, the breakdown of the spark-submit command is presented below to help you formulate your spark-submit command.
    
   **spark-submit** - This is the Apache Spark command to run a Spark application. **[Required]**
    
   **--class "org.apache.spark.spark_ml_titanis.classification"** - The class option for the spark-submit command tells Spark which class is the main class of the application. The classification class in the spark_ml_titanis package is the main entry-point of the application. For your own experiment, the package name and class name will vary, so use the package and class name you choose for the experiment instead. This is required to run any experiment. **[Required]**
    
   **--master local[\*]** - The master option for the spark-submit command tells Spark the mode to run the application in. The local[*] keyword indicates that Spark should run in local mode with all the available cores on the local machine. The asterisk symbol is used to use all the cores. If you would like to use a specific number of cores, it can be mentioned in the square brackets. In case you are running the experiment on a cluster, the URL for the cluster should be provided as the value for this option (e.g. spark://23.195.26.187:7077). When this option is omitted from the spark-submit command, Spark runs on local mode with the maximum number of available nodes on the local machine.
    
   **--jars ".\lib\moa.jar", ".\lib\titanis_2.13-0.1.jar"** - The jars option lets Spark know which external JARs we intend to use as dependencies for the application. Since this experiment uses the ADWIN Change drift detector from MOA and the StreamPrepareForLearning and MOADriftDetector classes from Titanis, the MOA and Titanis JAR file must be added to the spark-submit command to provide access to the MOA class. The JAR file can be saved anywhere in the project directory and the relative path to the files should be used here. **[Required]**
    
   **".\target\scala-2.13\spark-ml-titanis_2.13-0.1.0-SNAPSHOT.jar"** - The JAR file for the Spark project tells Spark which project to run. The JAR file for this experiment will be generated in this location after the build completes. Unless the default configuration for sbt is altered, the JAR file path will remain the same. The JAR file takes the name provided by the user in the sbt build file, so if running your own experiment, use the JAR file generated in the target folder in the project directory. **[Required]**
     
   **1>.\results.csv** - We output the results of the script to a file for convenience and future use. This part of the spark-submit command is optional, but recommended. For this experiment, the script outputs the trained model parameters and the instances where concept drift is detected to the Terminal, along with progress reports.
    
   **2>.\log.log** - We also save the logs of the Spark script to a file for inspection, in case there is an error. This is optional as well, but recommended, since the log output crowds the Terminal. It is easier to have the output of the script separate from the logs.

5. Run the formulated spark-submit command. If you are using the command provided, the Terminal will show no output since we are saving the output to a file.
6. Open the file where the output of the Terminal is being saved.
7. After some time, the file should show the trained Logistic Regression model parameters and the instances where drift was detected.
8. Since this experiment only uses one micro-batch of data, when it is processed, the file output should say "Finished analyzing batch 0". This indicates that the data has been processed and the application has now finished its task.

This concludes the concept drift detection from an external Spark application tutorial, and you can now run similar experiments using other Titanis and MOA concept drift detectors within your existing Spark pipelines.