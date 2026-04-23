# Canonical Spark Connect image for datalakeorm. Published as
# ghcr.io/caldempsey/lakeorm-spark-connect:<spark-version>.
#
# Owns the Spark/Iceberg/Delta/Hadoop-AWS/AWS-SDK version matrix
# centrally so every team adopting datalakeorm doesn't re-solve it.
# Getting any of these pins wrong produces ClassNotFoundException or
# NoSuchMethodError buried in Spark RPC stack traces — the kind of
# problem that eats a week of debugging.
#
# Version pins (Spark 4.0.x line, Scala 2.13):
#   Spark:         4.0.0
#   Iceberg:       1.10.0     (iceberg-spark-runtime-4.0_2.13)
#   Delta:         4.0.0      (delta-spark_2.13)
#   Hadoop AWS:    3.3.4
#   AWS SDK:       bundle aligned with Hadoop-AWS 3.3.4
#
# v0 scaffold — the Dockerfile body fills in as the image is tested
# against the chart and the docker-compose harness.

ARG SPARK_VERSION=4.0.0

FROM apache/spark:${SPARK_VERSION}

ARG ICEBERG_VERSION=1.10.0
ARG DELTA_VERSION=4.0.0
ARG HADOOP_AWS_VERSION=3.3.4
ARG AWS_SDK_VERSION=1.12.781

USER root

WORKDIR /opt/spark/jars-lakeorm

RUN curl -fsSL -o iceberg-spark-runtime-4.0_2.13-${ICEBERG_VERSION}.jar \
      https://repo1.maven.org/maven2/org/apache/iceberg/iceberg-spark-runtime-4.0_2.13/${ICEBERG_VERSION}/iceberg-spark-runtime-4.0_2.13-${ICEBERG_VERSION}.jar && \
    curl -fsSL -o delta-spark_2.13-${DELTA_VERSION}.jar \
      https://repo1.maven.org/maven2/io/delta/delta-spark_2.13/${DELTA_VERSION}/delta-spark_2.13-${DELTA_VERSION}.jar && \
    curl -fsSL -o hadoop-aws-${HADOOP_AWS_VERSION}.jar \
      https://repo1.maven.org/maven2/org/apache/hadoop/hadoop-aws/${HADOOP_AWS_VERSION}/hadoop-aws-${HADOOP_AWS_VERSION}.jar && \
    curl -fsSL -o aws-java-sdk-bundle-${AWS_SDK_VERSION}.jar \
      https://repo1.maven.org/maven2/com/amazonaws/aws-java-sdk-bundle/${AWS_SDK_VERSION}/aws-java-sdk-bundle-${AWS_SDK_VERSION}.jar && \
    cp ./*.jar /opt/spark/jars/

USER spark

EXPOSE 15002

# The chart + compose files pass --conf entries explicitly; the image
# just runs the connect server on 15002.
CMD ["/opt/spark/sbin/start-connect-server.sh", \
     "--conf", "spark.connect.grpc.binding.port=15002"]
