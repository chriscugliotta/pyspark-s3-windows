# pyspark-s3-windows

Apache Spark is fairly pleasant to use in a managed cloud environment like AWS EMR.  However, if you're trying to run Spark (with [S3A Connector](https://hadoop.apache.org/docs/current/hadoop-aws/tools/hadoop-aws/)) on a local Windows IDE?  Buckle up, and prepare to be frustrated.

```python
df = spark.read.csv('s3a://my-bucket/path/to/input/file.csv')
df.write.csv('s3a://my-bucket/path/to/output')
```

For _years_, I have tried (and failed) to run the code above on my local Windows machine, only to be taunted by a cacophony of obscure, 100-line Java stack traces.  After a while, I gave up and assumed it was not possible.  However, I recently stumbled upon stevel's detailed [post](https://stackoverflow.com/questions/44411493/java-lang-noclassdeffounderror-org-apache-hadoop-fs-storagestatistics/44500698#44500698) on StackOverflow, which inspired me to try again.  Lo and behold, it worked!

To help others (and my future self), I have carefully documented this arcane ritual.  Below, I will provide reproducible steps for installing PySpark with S3 connectivity on Windows.



## Contents

1.  [Dependencies](#dependencies)
2.  [Constraints](#constraints)
3.  [Instructions](#instructions)
4.  [Testing](#testing)
5.  [References](#references)



## Dependencies

This guide is intended for Python developers, so it assumes we will install Apache Spark indirectly via pip.  When pip installs PySpark, it collects most dependencies automatically, as seen in `.venv/Lib/site-packages/pyspark/jars`.  However, to enable the S3A connector, we must track down the following dependencies *manually*:

- JAR file:  `hadoop-aws`
- JAR file:  `aws-java-sdk-bundle`
- Executable:  `winutils.exe` (and `hadoop.dll`)



## Constraints

- Assuming we're installing Spark via pip, we can't pick the Hadoop version directly.  We can only pick the PySpark version, e.g. `pip install pyspark==3.1.3`, which will indirectly determine the Hadoop version.  For example, PySpark `3.1.3` maps to Hadoop `3.2.0`.

- All Hadoop JARs must have the _exact_ same version, e.g. `3.2.0`.  Verify this with `cd pyspark/jars && ls -l | grep hadoop`.  Notice that `pip install pyspark` automatically included some Hadoop JARs.  Thus, if these Hadoop JARs are `3.2.0`, then we should download `hadoop-aws:3.2.0` to match.

- `winutils.exe` must have the _exact_ same version as Hadoop, e.g. `3.2.0`.  Beware, winutils releases are [scarce](https://github.com/cdarlint/winutils).  Thus, we must carefully pick our PySpark/Hadoop version such that a matching winutils version exists.  Some PySpark/Hadoop versions _do not have_ a corresponding winutils release, thus they cannot be used!

- `aws-java-sdk-bundle` must be compatible with our `hadoop-aws` choice above.  For example, `hadoop-aws:3.2.0` depends on `aws-java-sdk-bundle:1.11.375`, which can be verified [here](https://mvnrepository.com/artifact/org.apache.hadoop/hadoop-aws/3.2.0).



## Instructions

With the above constraints in mind, here is a reliable algorithm for installing PySpark with S3A support on Windows:

1.  Find latest available version of `winutils.exe` [here](https://github.com/cdarlint/winutils).  At time of writing, it is `3.2.0`.  Place it at `C:/hadoop/bin`.  Set environment variable `HADOOP_HOME` to `C:/hadoop` and (important!) add `%HADOOP_HOME%/bin` to `PATH`.

2.  Find latest available version of PySpark that uses Hadoop version equal to above, e.g. `3.2.0`.  This can be determined by browsing PySpark's [`pom.xml`](https://github.com/apache/spark/blob/v3.1.3/pom.xml#L123) file across each release tag.  At time of writing, it is `3.1.3`.

3.  Find the version of `aws-java-sdk-bundle` that `hadoop-aws` requires.  For example, if we're using `hadoop-aws:3.2.0`, then we can use [this page](https://mvnrepository.com/artifact/org.apache.hadoop/hadoop-aws/3.2.0).  At time of writing, it is `1.11.375`.

4.  Create a venv and install the PySpark version from step 2.

```bash
python -m venv .venv
source .venv/Scripts/activate
pip install pyspark==3.1.3
```

5.  Download the AWS JARs into PySpark's JAR directory:

```bash
cd .venv/Lib/site-packages/pyspark/jars
ls -l | grep hadoop
curl -O https://repo1.maven.org/maven2/org/apache/hadoop/hadoop-aws/3.2.0/hadoop-aws-3.2.0.jar
curl -O https://repo1.maven.org/maven2/com/amazonaws/aws-java-sdk-bundle/1.11.375/aws-java-sdk-bundle-1.11.375.jar
```

6.  Download winutils:

```bash
cd C:/hadoop/bin
curl -O https://raw.githubusercontent.com/cdarlint/winutils/master/hadoop-3.2.0/bin/winutils.exe
curl -O https://raw.githubusercontent.com/cdarlint/winutils/master/hadoop-3.2.0/bin/hadoop.dll
```



## Testing

To verify your setup, try running the following script.

```python
import pyspark

spark = (pyspark.sql.SparkSession.builder
    .appName('my_app')
    .master('local[*]')
    .config('spark.hadoop.fs.s3a.access.key', 'secret')
    .config('spark.hadoop.fs.s3a.secret.key', 'secret')
    .getOrCreate())

# Test reading from S3.
df = spark.read.csv('s3a://my-bucket/path/to/input/file.csv')
print(df.head(3))

# Test writing to S3.
df.write.csv('s3a://my-bucket/path/to/output')
```
> You'll need to substitute your AWS keys and S3 paths, accordingly.

> If you recently updated your OS environment variables, e.g. `HADOOP_HOME` and `PATH`, you might need to close and re-open VSCode to reflect that.



## References

- stevel's answer on StackOverflow:  [link](https://stackoverflow.com/questions/44411493/java-lang-noclassdeffounderror-org-apache-hadoop-fs-storagestatistics/44500698#44500698).
