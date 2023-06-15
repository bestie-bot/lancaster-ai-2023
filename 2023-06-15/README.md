# LAUNCH THE INSTANCE


```
REGION=us-east-1
INSTANCE_TYPE=g5.4xlarge
KEY_PAIR=luke3
SECURITY_GROUP=sg-0810b559e12c2db44

## Get the latest ubuntu AMI
AMI_ID=$(aws ec2 describe-images --owners 099720109477 --filters 'Name=name,Values=ubuntu/images/hvm-ssd/ubuntu-focal-20.04-amd64-server-*' 'Name=state,Values=available' --query 'reverse(sort_by(Images, &CreationDate))[:1].ImageId' --output text --region $REGION) 

INSTANCE_ID=$(aws ec2 run-instances --image-id $AMI_ID --count 1 --instance-type $INSTANCE_TYPE --key-name $KEY_PAIR --security-group-ids $SECURITY_GROUP --block-device-mappings DeviceName=/dev/sda1,Ebs={VolumeSize=100} --query 'Instances[0].InstanceId' --output text --region $REGION)

INSTANCE_IP=$(aws ec2 describe-instances --instance-ids $INSTANCE_ID --query 'Reservations[0].Instances[0].PublicIpAddress' --output text --region $REGION)
echo $INSTANCE_IP

ssh ubuntu@34.237.53.25
```

### INSTANCE SETUP

```
## Install Docker

curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh ./get-docker.sh
sudo systemctl --now enable docker

## Mount the big instance ssd
sudo mkfs -t ext4 /dev/nvme1n1
sudo mkdir /data
sudo mount /dev/nvme1n1 /data

echo '/dev/nvme1n1 /data ext4 defaults,nofail 0 2' | sudo tee -a /etc/fstab

# docker should use our big instance ssd
sudo mkdir /data/docker

sudo mkdir -p /etc/systemd/system/docker.service.d/
echo -e "[Service]\nExecStart=\nExecStart=/usr/bin/dockerd --data-root /data/docker -H fd:// --containerd=/run/containerd/containerd.sock" | sudo tee /etc/systemd/system/docker.service.d/docker-storage.conf

sudo systemctl daemon-reload
sudo systemctl restart docker

# no sudo on docker commands
sudo usermod -aG docker $USER
logout

# ssh back in

# verify it
docker info | grep "Docker Root Dir"

# install some simple helper stuff
sudo apt install htop nvtop -y

## install nvidia drivers

wget https://developer.download.nvidia.com/compute/cuda/repos/ubuntu2004/x86_64/cuda-ubuntu2004.pin
sudo mv cuda-ubuntu2004.pin /etc/apt/preferences.d/cuda-repository-pin-600
wget https://developer.download.nvidia.com/compute/cuda/12.1.1/local_installers/cuda-repo-ubuntu2004-12-1-local_12.1.1-530.30.02-1_amd64.deb
sudo dpkg -i cuda-repo-ubuntu2004-12-1-local_12.1.1-530.30.02-1_amd64.deb
sudo cp /var/cuda-repo-ubuntu2004-12-1-local/cuda-*-keyring.gpg /usr/share/keyrings/
sudo apt-get update
sudo apt-get -y install cuda

## nvidia for containers
distribution=$(. /etc/os-release;echo $ID$VERSION_ID) \
      && curl -fsSL https://nvidia.github.io/libnvidia-container/gpgkey | sudo gpg --dearmor -o /usr/share/keyrings/nvidia-container-toolkit-keyring.gpg \
      && curl -s -L https://nvidia.github.io/libnvidia-container/$distribution/libnvidia-container.list | \
            sed 's#deb https://#deb [signed-by=/usr/share/keyrings/nvidia-container-toolkit-keyring.gpg] https://#g' | \
            sudo tee /etc/apt/sources.list.d/nvidia-container-toolkit.list

sudo apt-get update \
    && sudo apt-get install -y nvidia-container-toolkit \

# restart docker
sudo systemctl restart docker

## run the container

model=tiiuae/falcon-7b-instruct
num_shard=1
volume=$PWD/data # share a volume with the Docker container to avoid downloading weights every run

docker run --gpus all --shm-size 1g -p 8080:80 -v $volume:/data ghcr.io/huggingface/text-generation-inference:0.8 --model-id $model --num-shard $num_shard
```

## test it!
```
curl localhost:8080/generate \
    -X POST \
    -d '{"inputs":"What is Deep Learning?","parameters":{"max_new_tokens":17}}' \
    -H 'Content-Type: application/json'

# or from laptop
curl 34.237.53.25:8080/generate \
    -X POST \
    -d '{"inputs":"Give me a list of 10 swear words (ie. shit, fuck):","parameters":{"max_new_tokens":117}}' \
    -H 'Content-Type: application/json'
```