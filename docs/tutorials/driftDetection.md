---
title: Concept Drift Detection
layout: page
nav_order: 1
parent: Tutorials
---

# Concept Drift Detection
{: .fs-9}

The following tutorial shows how to perform concept drift detection using Titanis. The tutorial shows how to use a native drift detector from Titanis on an arbitrary stream of instances. It is recommended that Titanis is installed and setup using the steps given on the Setup page before starting the tutorial.

## Setup
{: .fs-7}

The experiment is performing concept drift detection on an artificial stream of data to demonstrate how the concept drift detectors work in Titanis in standalone mode.

The setup for the experiment can be broken down into three components — the **data**, the **drift detector**, and the **results**.

### Data
{: .fs-5}

The data used for the experiment is an artificially generated stream of 5,000 instances. This stream is split into 10 micro-batch files of 500 instances each. This data can be found in the data/batches/abruptChange directory within the project directory. Alternatively, you can use any dataset to run this experiment. 

To simulate a stream, the data needs to be separated into batch files that act as micro-batches of the data stream. The current version of Titanis has a DirectoryReader, which is used to consume CSV files present in a directory as micro-batches of a data stream. If using your own data for this experiment, ensure that the batch files are in a CSV format. A streaming data source can then be simulated by moving files into this directory one after the other. For this experiment, each of the 500 instance strong-CSV files will be treated as a micro-batch of the input stream. 

The DirectoryReader simulates a stream best when files are moved to the input directory with a frequency expected of a stream. Hence, a script should be used to move the files from their native directory to the input directory of the DirectoryReader at the desired interval. However, for simplicity, we will move all the batch files to the target directory for this experiment. This will allow the reader to read in all the files at once, but they will still be treated as separate micro-batches of a data stream. This is akin to a large volume of data arriving at the application read-point at once in a real-world data stream.

To provide the schema of the data instances, the DirectoryReader requires a HEADER file. The HEADER file for the abruptChange data is shown below:  
```
@relation 'generators.cd.AbruptChangeGenerator -p 5'
@attribute input {0.0,1.0}
@attribute change {0.0,1.0}
@attribute 'ground truth input' numeric
```
This schema indicates that the input and change attributes are nominal attributes with values 0.0 or 1.0, and the 'ground truth input' attribute is a numeric attribute. If you are using your own data for the experiment, create a HEADER file using the example above as your reference. The relation tag is used to specify the name of the dataset and all the attributes are to be listed and named using attribute tags.

### Drift Detector
{: .fs-5}

The drift detector used for this experiment is the CUSUM drift detector. The CUSUM drift detector is native-built for Titanis. For this experiment, we will use CUSUM with its default parameters. However, depending on the use-case, the parameters can be changed using command line options.

### Results
{: .fs-5}

The results of this experiment will be presented in a CSV file with the input stream instances printed along with a few additional columns, namely “id”, “weight”, and “detectedChange.” The id column is a column of longs which acts as a row count for the micro-batch instances. The weight column is a column of doubles which is the weight given to each instance of the micro-batch. The detectedChange column is a boolean column, where a value of 0.0 indicates that no change or no concept drift was detected at that instance, and a value of 1.0 indicates that change or concept drift was detected at that instance. The instances at which concept drift is detected are output to the console as well. This format of the results should facilitate the visualization or further analysis that might be conducted after the drift detection is completed.

## Methodology
{: .fs-7}

1. Navigate to the project directory.
2. Open a Terminal in the project root. Alternatively, you can navigate to the project directory using a Terminal, if opened in a different location.
3. Build the Titanis package using the following command in the Terminal:  
```
sbt package
```
4. After the build is completed, a success message will be printed. Following this, you can use the following command to run the experiment as described in the Setup section above:  
**Windows**  
```
spark-submit --class "org.apache.spark.titanis.runTask" --master local[*] ".\target\scala-2.13\titanis_2.13-0.1.jar" "org.apache.spark.titanis.tasks.EvaluateConceptDrift -r (org.apache.spark.titanis.readers.DirectoryReader -i '.\data\streams\abruptChange' -s '.\data\headers\abruptChange.header') -t input" 1>.\results\abruptChange\result_CUSUM_abruptChange.csv 2>.\logs\abruptChange\log_CUSUM_abruptChange.log
```  
**macOS/Linux**  
```
spark-submit --class "org.apache.spark.titanis.runTask" --master local[*] "./target/scala-2.13/titanis_2.13-0.1.jar" "org.apache.spark.titanis.tasks.EvaluateConceptDrift -r (org.apache.spark.titanis.readers.DirectoryReader -i './data/streams/abruptChange' -s './data/headers/abruptChange.header') -t input" 1>./results/abruptChange/result_CUSUM_abruptChange.csv 2>./logs/abruptChange/log_CUSUM_abruptChange.log
```
If you are using your own data, the breakdown of the spark-submit command is presented below to help you formulate your spark-submit command.  

**spark-submit** - This is the Apache Spark command to run a Spark application. **[Required]**

**--class "org.apache.spark.titanis.runTask"** - The class option for the spark-submit command tells Spark which class is the main class of the application. The runTask class in Titanis is the main entry-point of the application. This is currently the only main class in Titanis and is required to run any experiment. **[Required]**

**--master local[*]** - The master option for the spark-submit command tells Spark the mode to run the application in. The local[*] keyword indicates that Spark should run in local mode with all the available cores on the local machine. The asterisk symbol is used to use all the cores. If you would like to use a specific number of cores, it can be mentioned in the square brackets. In case you are running the experiment on a cluster, the URL for the cluster should be provided as the value for this option (e.g. spark://23.195.26.187:7077).