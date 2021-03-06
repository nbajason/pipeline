FROM fluxcapacitor/package-ubuntu-16.04:master

WORKDIR /root

ENV \
  TERM=xterm

###################
# Setup OpenJDK 1.8
###################
RUN \
  apt-get update \
  && apt-get install -y software-properties-common \
  && add-apt-repository -y ppa:openjdk-r/ppa \
  && apt-get update \
  && apt-get install -y --no-install-recommends openjdk-8-jdk openjdk-8-jre-headless \
  && apt-get install -y apt-transport-https \
  && apt-get install -y wget \
  && apt-get clean \
  && rm -rf /var/lib/apt/lists/*

ENV \
  JAVA_HOME=/usr/lib/jvm/java-8-openjdk-amd64/

ENV \
  SHELL=/bin/bash

RUN \
  rm /bin/sh \
  && ln -s /bin/bash /bin/sh

RUN \
  echo "deb https://dl.bintray.com/sbt/debian /" | tee -a /etc/apt/sources.list.d/sbt.list \
  && apt-key adv --keyserver hkp://keyserver.ubuntu.com:80 --recv 2EE0EA64E40A89B84B2DF73499E82A75642AC823 \
  && apt-get update \
  && apt-get install -y sbt

# Install Anaconda with Python3
RUN wget -q https://repo.continuum.io/miniconda/Miniconda3-4.3.21-Linux-x86_64.sh -O /tmp/miniconda.sh  && \
    echo 'c1c15d3baba15bf50293ae963abef853 */tmp/miniconda.sh' | md5sum -c - && \
    bash /tmp/miniconda.sh -f -b -p /opt/conda && \
    /opt/conda/bin/conda install --yes python=3.6 sqlalchemy tornado jinja2 traitlets requests pip && \
    /opt/conda/bin/pip install --upgrade pip && \
    rm /tmp/miniconda.sh

ENV \
  PATH=/opt/conda/bin:$PATH

RUN \
  pip install --upgrade pip

# Anaconda's libgfortran=3.0 is not co-operating, so we use apt-get
RUN \
  apt-get install -y libgfortran3

RUN \
  conda install -c anaconda openblas

RUN \
  apt-get install -y nginx 

RUN \
  service nginx start

COPY config/ config/

RUN \
  mv /etc/nginx/sites-available/default /etc/nginx/sites-available/default.orig \
  && cd /etc/nginx/sites-available/ \
  && ln -s /root/config/nginx/default \
  && cd /etc/nginx/sites-enabled/ \
  && rm default \
  && ln -s /etc/nginx/sites-available/default

ENV \
  PROMETHEUS_VERSION=1.7.1

RUN \
  wget https://github.com/prometheus/prometheus/releases/download/v$PROMETHEUS_VERSION/prometheus-$PROMETHEUS_VERSION.linux-amd64.tar.gz \
  && tar xvfz prometheus-$PROMETHEUS_VERSION.linux-amd64.tar.gz \
  && rm prometheus-$PROMETHEUS_VERSION.linux-amd64.tar.gz

RUN \
  mv /root/prometheus-$PROMETHEUS_VERSION.linux-amd64/prometheus.yml /root/prometheus-$PROMETHEUS_VERSION.linux-amd64/prometheus.yml.orig \
  && cd /root/prometheus-$PROMETHEUS_VERSION.linux-amd64/ \
  && ln -s /root/config/prometheus/prometheus.yml

ENV \
  PATH=/root/prometheus-$PROMETHEUS_VERSION.linux-amd64/:$PATH

ENV \
  GRAFANA_VERSION=4.4.3

RUN \
  wget https://s3-us-west-2.amazonaws.com/grafana-releases/release/grafana-$GRAFANA_VERSION.linux-x64.tar.gz \ 
  && tar -zxvf grafana-$GRAFANA_VERSION.linux-x64.tar.gz \
  && rm grafana-$GRAFANA_VERSION.linux-x64.tar.gz

ENV \
  PATH=/root/grafana-$GRAFANA_VERSION/bin:$PATH 

RUN \
  cd /root/grafana-$GRAFANA_VERSION/conf \
  && ln -s /root/config/grafana/custom.ini \
  && ln -s /root/config/grafana/dashboards

RUN \
  mkdir -p /root/logs

ENV \
  LOGS_HOME=/root/logs

COPY sysutils/ sysutils/

ENV \
  HYSTRIX_DASHBOARD_PORT=7979

COPY dashboard/ dashboard/

RUN \
  cd dashboard/hystrix-dashboard && ../gradlew --no-daemon buildAllProducts assemble -x test

RUN \
  sbt -mem 1500 cleanFiles

COPY jvm/ jvm/

ENV \
  PIPELINE_JVM_MODEL_SERVER_PATH=/root/jvm

RUN \
  cd $PIPELINE_JVM_MODEL_SERVER_PATH && sbt -mem 1500 package

ENV \
  CONFLUENT_VERSION=3.3.0 \
  CONFLUENT_MAJOR_VERSION=3.3

ENV \
  CONFLUENT_HOME=/root/confluent-${CONFLUENT_VERSION}

ENV \
  PATH=$CONFLUENT_HOME/bin:$PATH

RUN \
 wget http://packages.confluent.io/archive/${CONFLUENT_MAJOR_VERSION}/confluent-oss-${CONFLUENT_VERSION}-2.11.tar.gz \
 && tar xvzf confluent-oss-${CONFLUENT_VERSION}-2.11.tar.gz \
 && rm confluent-oss-${CONFLUENT_VERSION}-2.11.tar.gz

RUN \
  git clone https://github.com/edenhill/librdkafka.git \
  && cd librdkafka \
  && ./configure \
  && make \
  && make install

ENV \
  PIPELINE_WEBSOCKET_KAFKA_SERVER_PATH=/root/src/main/python/kafka

ENV \
  PIPELINE_WEBSOCKET_KAFKA_SERVER_PORT=5959

ENV \
  PIPELINE_WEBSOCKET_KAFKA_SERVER_UPDATE_INTERVAL=1000

# Must run ths or you will see the following error:
#   ImportError: librdkafka.so.1: cannot open shared object file: No such file or directory
RUN \
  ldconfig

ENV \
  PIPELINE_MODEL_SERVER_PATH=/root/src/main/python/model_server

ENV \
  PIPELINE_MODEL_SERVER_PORT=9876

ENV \
  PIPELINE_MODEL_SERVER_PROMETHEUS_PORT=10254

ENV \
  PIPELINE_MODEL_SERVER_TENSORFLOW_SERVING_PORT=9000

ENV \
  PIPELINE_MODEL_SERVER_TENSORFLOW_SERVING_REQUEST_BATCHING=true

RUN \
  echo "" \
  && echo "Installing TensorFlow Serving..." \
  && echo "deb [arch=amd64] http://storage.googleapis.com/tensorflow-serving-apt stable tensorflow-model-server tensorflow-model-server-universal" | tee /etc/apt/sources.list.d/tensorflow-serving.list \
  && curl https://storage.googleapis.com/tensorflow-serving-apt/tensorflow-serving.release.pub.gpg | apt-key add - \
  && apt-get update \
  && apt-get install -y tensorflow-model-server;

RUN \
  pip install git+https://github.com/wintoncode/winton-kafka-streams.git

ENV \
  TENSORBOARD_VERSION=0.1.5 \
  TENSORBOARD_LOGDIR=$LOGS_HOME/tensorboard

RUN \
  pip install tensorflow-tensorboard==$TENSORBOARD_VERSION

ENV \
  TF_CPP_MIN_LOG_LEVEL=0

# Drop Server
#RUN \
#  wget https://storage.googleapis.com/kubernetes-release/release/v1.7.5/bin/linux/amd64/kubectl \
#  && chmod a+x kubectl \
#  && mv kubectl /usr/local/bin/kubectl
# Drop : Docker in Docker
#RUN \
#  curl -fsSL get.docker.com -o get-docker.sh \
#  && sh get-docker.sh

COPY html/ /var/www/html/
COPY run run

ARG model_type
RUN \
  echo $model_type
ENV \
  PIPELINE_MODEL_TYPE=$model_type

ARG model_name
RUN \
  echo $model_name
ENV \
  PIPELINE_MODEL_NAME=$model_name

ARG model_chip
RUN \
  echo $model_chip
ENV \
  PIPELINE_MODEL_CHIP=$model_chip

RUN \
  cd $PIPELINE_JVM_MODEL_SERVER_PATH/lib/jni \
  && ln -s libtensorflow_jni-$PIPELINE_MODEL_CHIP.so libtensorflow_jni.so

ARG model_tag
RUN \
  echo $model_tag
ENV \
  PIPELINE_MODEL_TAG=$model_tag

# TODO:  model_path (and copy) is not required for drop server
ARG model_path
RUN \
  echo $model_path
ENV \
  PIPELINE_MODEL_PATH=/root/model/$model_type/$model_name/$model_tag

COPY $model_path $PIPELINE_MODEL_PATH

# TODO:  This is only required for Drop because Drop builds new models upon upload
#COPY / drop/

#ENV \
#  PIPELINE_DROP_SERVER_PATH=/root/src/main/python/drop

#ENV \
#  PIPELINE_DROP_PATH=/root/drop
#RUN \
#  mkdir -p $PIPELINE_DROP_PATH

#ENV \
#  PIPELINE_DROP_SERVER_PORT=9877

ENV \
  PIPELINE_CONDA_ENV_NAME=pipeline-$model_type-$model_name-$model_tag

COPY scripts/ scripts/
COPY src/ src/
RUN \
  /root/scripts/setup-environment.sh 

RUN \
  /root/scripts/train-model-if-requested.sh

# Bazel (Guild)
#RUN \
#  echo "deb [arch=amd64] http://storage.googleapis.com/bazel-apt stable jdk1.8" | tee /etc/apt/sources.list.d/bazel.list \
#  && curl https://bazel.build/bazel-release.pub.gpg | apt-key add - \
#  && apt-get update \
#  && apt-get install -y bazel

# Guild
#RUN \
#  conda install python=2.7.10 \
#  pip2 install psutil

# Guild
#RUN \
#  git clone https://github.com/guildai/guild-python.git \
#  && cd guild-python \
#  && bazel build guild 

# Guild
#ENV \
#  PATH=/root/guild-python/bazel-bin/guild:$PATH \
#  PYTHONPATH=/root/guild-python/:$PYTHONPATH

# Guild
#RUN \
#  guild check --tests

RUN \
  pip install pipeline-ai-cli==0.71

# Don't forget to update the pipeline cli if these ports change!
EXPOSE \
  # Nginx (80 is too common and could clash with local development) \
  6969 \
  # Python-based Model Server (Primary Model) \
  9876 \
  # TensorFlow-based Model Server (Primary Model) \
  9000 \
  # JVM-based Model Server (Primary Model) \
  9040 \
  # Prometheus Metrics Server \
  9090 \
  # Grafana Dashboards \
  3000 \
  # Kafka Broker \
  9092 \
  # Kafka REST API \
  8082 \
  # Kafka Schema Manager \
  8081 \
  # ZooKeeper \
  2181 \
  # WebSocket-based Kafka Consumer \
  5959 \
  # TensorBoard \
  6006 \
  # TensorFlow Training UI (Guild) \
  6333 \
  # Drop Server \
  9877 \
  # Hystrix Dashboard \
  7979 \
  # Prometheus Endpoint (Python)
  10254

CMD ["supervise", "."]
