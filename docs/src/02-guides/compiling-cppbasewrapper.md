# Compiling CppBaseWrapper

sudo apt install libsasl2-dev
# Requires Confluent APT
sudo apt install librdkafka-dev

mkdir build
cd build
cmake ..
# cmake .. --fresh
make