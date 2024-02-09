---
title: Setup
layout: page
nav_order: 2
---

# Download Titanis
{: .fs-9 }

Titanis is an open-source library available for download through Github. You can use this link to the repository to clone it or download the ZIP file with the code.

# Setup Titanis
{: .fs-9}

Before being able to run Titanis, there are some prerequisites that you need to install or check whether you have installed on your machine.

## Prerequisites
{: .fs-7}

- **Apache Spark 3.4.1** - Apache Spark version 3.4.1 is recommended for using Titanis. The current version of Titanis is built using Spark 3.4.1 and other versions of Spark might have changes that are not compatible with Titanis. Future versions of Titanis will be built for the latest version of Spark available at that time. It is recommended that the Spark version pre-built for Apache Hadoop 3.3 and later with Scala 2.13 is used to run Titanis.

- **Apache Hadoop 3.3** - Apache Hadoop version 3.3 is recommended for using Apache Spark 3.4.1 and Titanis. Apache Spark uses the Hadoop Distributed File System and depends on Hadoop win-utils on the machine to run. If you are using a Windows machine, some versions of Hadoop have had some issues with installation. It is recommended that you download the win-utils file for the Hadoop version in this repository and set the directory with the win-utils as your HADOOP_HOME environment variable. A full guide on how to setup Apache Spark and Apache Hadoop on Windows is available here.

- **Scala 2.13** - Titanis is built using Scala 2.13. Different versions of Scala might comprise changes that are not handled appropriately in Titanis. This can lead to unforeseen runtime errors. Hence, it is recommended to install Scala 2.13 for Titanis.

- **sbt** - sbt is the build tool for Scala. It is required to build the Titanis project before it can run. sbt is installed in conjunction with Scala using the downloaders on their official website.

- **(Optional) IntelliJ Idea Community Edition** - IntelliJ Idea is an IDE that can be used to navigate the codebase by developers and users who are interested in examining the code. This IDE is preferred because it offers a Scala plugin that assists in auto-completion and syntax-error detection. Another IDE with a Scala plugin is Visual Studio Code. This is an optional requirement to run Titanis.

# Run Titanis
{: .fs-9}

Let us use an example script from the repository to demonstrate how to run Titanis.

1. Navigate to the Titanis project directory.
2. Open a Terminal in the directory. Alternatively, you can navigate to the project directory from within the Terminal, if opened in a different location.
3. Build the project using the sbt command:  
```
sbt package
```
4. Once the build process is completed, Titanis is ready to run an experiment. The experiment can be run using one of the provided scripts in the scripts directory of the project. Here, we will run the kNNClassification script using the following command in the Terminal:  
**Windows**  
```
.\scripts\kNNClassification.bat
```  
**macOS/Linux**  
```
./scripts/kNNClassification.sh
```  
Alternatively, you can run an experiment directly from the Terminal by pasting the following command:  
**Windows**  
```
spark-submit --class "org.apache.spark.titanis.runTask" --master local[*] ".\target\scala-2.13\titanis_2.13-0.1.jar" "org.apache.spark.titanis.tasks.EvaluateLearner -r (org.apache.spark.titanis.readers.DirectoryReader -i '.\data\streams\elecNormNew' -s '.\data\headers\elecNormNew.header') -e (org.apache.spark.titanis.evaluators.BasicClassificationEvaluator -f)" 1>.\results\elecNormNew\result_kNN_elecNormNew.csv 2>.\logs\elecNormNew\log_kNN_elecNormNew.log
```  
**macOS\Linux**  
```
spark-submit --class "org.apache.spark.titanis.runTask" "./target/scala-2.13/titanis_2.13-0.1.jar" "org.apache.spark.titanis.tasks.EvaluateLearner -r (org.apache.spark.titanis.readers.DirectoryReader -i './data/streams/elecNormNew' -s './data/headers/elecNormNew.header') -e (org.apache.spark.titanis.evaluators.BasicClassificationEvaluator -f)" 1>./results/elecNormNew/result_kNN_elecNormNew.csv 2>./logs/elecNormNew/log_kNN_elecNormNew.log
```
5. After running the command, the experiment begins running. Considering this is a test experiment, all the data files are provided in the target directory for the script. Each file of the data will be treated as a micro-batch of a simulated data stream. Titanis will read the stream from the target directory and output the results of the experiment to the batch_results directory in the project directory.
6. Navigate to the batch_results directory.
7. Navigate to the evaluations sub-directory.
8. The results of the evaluation task will be output to this directory after each micro-batch is processed completely. The files in this directory are CSV files, and the evaluation metrics for the performance of the model on a given micro-batch of data will be printed in these files in a CSV format.