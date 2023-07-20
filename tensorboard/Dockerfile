FROM pytorch/pytorch:1.13.0-cuda11.6-cudnn8-devel

RUN apt-get update &&\
    apt-get -y upgrade &&\
    apt-get install -y git net-tools vim sudo tcsh gcc g++ unzip python3 python3-pip &&\
    apt-get clean &&\
    rm -rf /var/lib/apt/lists/*

RUN pip3 --no-cache-dir install torch \
    torchvision \
    torchaudio \
    tensorboard \
    jupyterlab \
    jupyter

CMD ["tensorboard", "--logdir", "./logs", "--host=0.0.0.0", "--port=8324"]