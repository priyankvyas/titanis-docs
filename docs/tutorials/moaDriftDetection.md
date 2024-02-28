---
title: Concept Drift Detection using MOA
layout: page
nav_order: 2
parent: Tutorials
---

# Concept Drift Detection using MOA
{: .fs-9}

The following tutorial shows how to perform concept drift detection with MOA drift detectors using Titanis. The tutorial shows how to use a drift detector from MOA from within Titanis on an arbitrary stream of instances. It is recommended that Titanis is installed and setup using the steps given on the Setup page before starting the tutorial.

{: .note } The tutorials for Concept Drift Detection and Concept Drift Detection using MOA are highly similar and serve as standalone tutorials. Please choose only one of the tutorials to follow methodically to avoid repeating steps.

## Setup
{: .fs-7}

The experiment is performing concept drift detection on an artificial stream of data to demonstrate how the MOA concept drift detectors work in Titanis in standalone mode.

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

The drift detector used for this experiment is the ADWIN Change drift detector from MOA. The ADWIN Change drift detector is native-built in Java for the MOA machine learning library. However, Titanis enables the import of learners and drift detectors from MOA, using wrappers built in Titanis. For this experiment, we will use ADWIN Change with its default parameters. However, depending on the use-case, the parameters can be changed using command line options.

### Results
{: .fs-5}

The results of this experiment will be presented in a CSV file with the input stream instances printed along with a few additional columns, namely “id”, “weight”, and “detectedChange.” The id column is a column of longs which acts as a row count for the micro-batch instances. The weight column is a column of doubles which is the weight given to each instance of the micro-batch. The detectedChange column is a boolean column, where a value of 0.0 indicates that no change or no concept drift was detected at that instance, and a value of 1.0 indicates that change or concept drift was detected at that instance. The instances at which concept drift is detected are output to the console as well. This format of the results should facilitate the visualization or further analysis that might be conducted after the drift detection is completed.

## Methodology
{: .fs-7}

1. Navigate to the project directory.
2. Open a Terminal in the project root. Alternatively, you can navigate to the project directory using a Terminal, if opened in a different location.
3. Build the Titanis package using the following command in the Terminal:
```shell
sbt package
```
4. After the build is completed, a success message will be printed. Following this, you can use the following command to run the experiment as described in the Setup section above:  
   **Windows**
```shell
spark-submit --class "org.apache.spark.titanis.runTask" --master local[*] --jars “.\lib\moa.jar” ".\target\scala-2.13\titanis_2.13-0.1.jar" "org.apache.spark.titanis.tasks.EvaluateConceptDrift -d (org.apache.spark.titanis.driftDetectors.MOADriftDetector -D moa.classifiers.core.driftdetection.ADWINChangeDetector) -r (org.apache.spark.titanis.readers.DirectoryReader -i '.\data\streams\abruptChange' -s '.\data\headers\abruptChange.header') -t input" 1>.\results\abruptChange\result_ADWIN_abruptChange.csv 2>.\logs\abruptChange\log_ADWIN_abruptChange.log
```  
**macOS/Linux**
```shell
spark-submit --class "org.apache.spark.titanis.runTask" --master local[*] --jars “./lib/moa.jar” "./target/scala-2.13/titanis_2.13-0.1.jar" "org.apache.spark.titanis.tasks.EvaluateConceptDrift -d (org.apache.spark.titanis.driftDetectors.MOADriftDetector -D moa.classifiers.core.driftdetection.ADWINChangeDetector) -r (org.apache.spark.titanis.readers.DirectoryReader -i './data/streams/abruptChange' -s './data/headers/abruptChange.header') -t input" 1>./results/abruptChange/result_ADWIN_abruptChange.csv 2>./logs/abruptChange/log_ADWIN_abruptChange.log
```
If you are using your own data, the breakdown of the spark-submit command is presented below to help you formulate your spark-submit command.

**spark-submit** - This is the Apache Spark command to run a Spark application. **[Required]**

**--class "org.apache.spark.titanis.runTask"** - The class option for the spark-submit command tells Spark which class is the main class of the application. The runTask class in Titanis is the main entry-point of the application. This is currently the only main class in Titanis and is required to run any experiment. **[Required]**

**--master local[\*]** - The master option for the spark-submit command tells Spark the mode to run the application in. The local[*] keyword indicates that Spark should run in local mode with all the available cores on the local machine. The asterisk symbol is used to use all the cores. If you would like to use a specific number of cores, it can be mentioned in the square brackets. In case you are running the experiment on a cluster, the URL for the cluster should be provided as the value for this option (e.g. spark://23.195.26.187:7077). When this option is omitted from the spark-submit command, Spark runs on local mode with the maximum number of available nodes on the local machine.

**--jars ".\lib\moa.jar"** - The jars option lets Spark know which external JARs we intend to use as dependencies for the application. Since this experiment uses the ADWIN Change drift detector from MOA, the MOA JAR file must be added to the spark-submit command to provide access to the MOA class. The JAR file is included in the project repository and should be available at the same path shown in the command above. **[Required]**

**".\target\scala-2.13\titanis_2.13-0.1.jar"** - The JAR file for the Spark project tells Spark which project to run. The JAR file for Titanis will be generated in this location after the build completes. Unless the default configuration for sbt is altered, the JAR file path will remain the same. **[Required]**

**"org.apache.spark.titanis.tasks.EvaluateConceptDrift -d (org.apache.spark.titanis.driftDetectors.MOADriftDetector -D moa.classifiers.core.driftdetection.ADWINChangeDetector) -r (org.apache.spark.titanis.readers.DirectoryReader -i '.\data\streams\abruptChange' -s '.\data\headers\abruptChange.header') -t input"** - After telling Spark which application to run, we provide Spark any arguments that the application will require. The runTask main class for Titanis requires command line arguments to specify the Task to run. In the event that these arguments are omitted, Titanis throws an error, prompting the user to specify the Task they want to run. If you would like to run your own experiment the argument will be slightly different to what it is now, depending on the experiment. To further explain how to formulate the command line arguments for your experiment, the breakdown of the command line arguments is provided below. **[Required]**

**org.apache.spark.titanis.tasks.EvaluateConceptDrift** - runTask only requires one argument i.e., the Task to run. This argument specifies the classpath of the Task chosen for this experiment. There are some cases where a shortened classpath also works, but it is recommended to use the full classpath for consistency. The Tasks are present in the tasks directory of the project. There are currently two Tasks available to choose from &#8212; EvaluateConceptDrift and EvaluateLearner. This experiment aims to perform concept drift detection, so we use the former. If you wish to evaluate a learning algorithm from Titanis or MOA, the latter should be used. The Tasks themselves are configurable to suit your experimental needs. They can be configured using command line options. The range of options available can be found in the documentation for the specific Task. We will explain a few of them that are used for this experiment below. **[Required]**

**-d (org.apache.spark.titanis.driftDetectors.MOADriftDetector -D moa.classifiers.core.driftdetection.ADWINChangeDetector)** - The -d option is used to specify the drift detector to be used for the Task. The default value for the option is the native CUSUM drift detector. There are currently two Titanis-native drift detectors available &#8212; CUSUM and Page-Hinckley. Apart from the native drift detectors, all the drift detectors from the MOA library can be used with the wrapper, MOADriftDetector. The MOADriftDetector, like the other Titanis Drift Detectors is configurable using options. The option for the MOADriftDetector is explained below.

**-D moa.classifiers.core.driftdetection.ADWINChangeDetector** - The -D option for the MOADriftDetector is used to specify which drift detector from MOA to use. There are several available drift detectors in the MOA machine learning library. If you wish to run an experiment with a different drift detector, you can inspect the JAR file for MOA, provided with the repository, to identify the classpath for the desired drift detector. The default value for this option is the CusumDM detector from MOA. For this experiment, we have chosen to use the ADWINChangeDetector from MOA, with its default parameters.

**-r (org.apache.spark.titanis.readers.DirectoryReader -i '.\data\streams\abruptChange' -s '.\data\headers\abruptChange.header')** - The -r option is used to specify the Reader for the Task. Currently, Titanis includes only one Reader i.e., the DirectoryReader. As explained in the Setup section previously, the DirectoryReader observes a given target directory and when files are moved into the target directory, they are treated as micro-batches of a data stream and read into the Spark application. The DirectoryReader is further configurable to allow users to specify the path to the target directory and the path to the schema file for the input data. The default Reader used is the DirectoryReader. However, there are no default values for the target directory and schema file options, so it is mandatory to include this option in your spark-submit command. More information about the options for DirectoryReader are provided below. **[Required]**

**-i '.\data\streams\abruptChange'** - The -i option is used to specify the input or target directory for the DirectoryReader. The value for this option should be the file path to the directory where the stream data will arrive or is already present. This experiment uses a sample dataset that is present in the project directory. In case you are running your own experiment, the file path should be altered to the location where your data is or will be. **[Required]**

**-s '.\data\headers\abruptChange.header')** - The -s option is used to specify the schema file for the input data. The schema file should be of type HEADER. An example of what the HEADER file looks like is provided above in the Setup section. In case you are using your own data, ensure that schema for the input data is formatted in the HEADER file format to avoid any runtime errors. The schema file in use for this experiment is provided with the data. **[Required]**

**-t input** - The -t option is used to specify the name of the target attribute. The default name for the target attribute is 'class'. If this option is not provided in the command line argument, the default value will be used. For this data, we would like to treat the 'input' field as the target attribute, hence the value is set to 'input'. In case you are using your own data, check the schema of the input data and use the right attribute name to avoid runtime errors or undesired results.

**1>.\results\abruptChange\result_ADWIN_abruptChange.csv** - We output the results of the script to a file for convenience and future use. This part of the spark-submit command is optional, but recommended. For the EvaluateConceptDrift Task, the script outputs instances where concept drift is detected to the Terminal, along with progress reports. The EvaluateLearner Task does not output anything except for progress reports.

**2>.\logs\abruptChange\log_ADWIN_abruptChange.log** - We also save the logs of the Spark script to a file for inspection, in case there is an error. This is optional as well, but recommended, since the log output crowds the Terminal. It is easier to have the output of the script separate from the logs.

5. Run the formulated spark-submit command. If you are using the command provided, the Terminal will show no output since we are saving the output to a file.
6. Open the file where the output of the Terminal is being saved.
7. After some time, the file should show the instance where drift was detected.
8. Once one micro-batch of data is processed, the file output should say "Finished analyzing batch 0". This indicates that the first batch has been processed and the application is now processing the next batch.

This concludes the concept drift detection using MOA drift detectors tutorial, and you can now run similar experiments using other Titanis and MOA concept drift detectors.