# nvd2系インスタンスにNvidia HPC SDKと他数値計算ライブラリ群を導入する
## 目的
1. OpenACCの有効化(GPU)
2. FTTW3の動作(CPU)
3. ScaLAPACKの動作(CPU)

## Nvidi HPC SDKの導入
```bash
cd
sudo wget https://developer.download.nvidia.com/hpc-sdk/20.11/nvhpc-20-11_20.11_amd64.deb
sudo wget https://developer.download.nvidia.com/hpc-sdk/20.11/nvhpc-2020_20.11_amd64.deb 
sudo wget https://developer.download.nvidia.com/hpc-sdk/20.11/nvhpc-20-11-cuda-multi_20.11_amd64.deb
sudo apt-get install ./nvhpc-20-11_20.11_amd64.deb ./nvhpc-2020_20.11_amd64.deb ./nvhpc-20-11-cuda-multi_20.11_amd64.deb
```
## makeとvimの導入
```bash
sudo apt update
sudo apt install -y make vim
```

## ScaLAPACKの導入
### BLAS(v3.8.0)の導入
```bash
cd
wget http://www.netlib.org/blas/blas.tgz
tar -zxvf blas.tgz
mv  BLAS-3.8.0/ BLAS/
cd BLAS/
gfortran -O3 -std=legacy -m64 -fno-second-underscore -fPIC -c *.f
ar r libfblas.a *.o
ranlib libfblas.a
rm -rf *.o
export BLAS=~/src/BLAS/libfblas.a
ln -s libfblas.a libblas.a
mv ~/BLAS /usr/local/
```

### LAPACK(v3.4.0)の導入
```bash
cd
wget http://www.netlib.org/lapack/lapack-3.4.0.tgz
tar -zxvf lapack-3.4.0.tgz
cd lapack-3.4.0.tgz
cp INSTALL/make.inc.gfortran make.inc
## make.incをhttps://thelinuxcluster.com/2012/04/09/building-lapack-3-4-with-intel-and-gnu-compiler/ に従って編集すること
make lapacklib
make clean
mkdir -p /usr/local/lapack
cp liblapack.a 
mv liblapack.a /usr/local/lapack/
export LAPACK=/usr/local/lapack/liblapack.a
```

### OpenMPI(v3.1.3)の導入
```bash
cd
wget https://download.open-mpi.org/release/open-mpi/v3.1/openmpi-3.1.3.tar.gz --no-check-certificate
gunzip -c openmpi-3.1.3.tar.gz | tar xf -
cd openmpi-3.1.3
./configure --prefix=/opt/openMPI CC=gcc CXX=g++ F77=gfortran FC=gfortran
make
sudo make install
## .bashrcをhttps://qiita.com/kitarow0309/items/8bb6fe2006760ed5a72e に従って編集する
source ~/.bashrc
```

### ScaLAPACK(v2.0.2)の導入
```bash
cd
wget http://www.netlib.org/scalapack/scalapack-2.0.2.tgz
tar -zxvf scalapack-2.0.2.tgz
cd scalapack-2.0.2
cp SLmake.inc.example SLmake.inc
## SLmakeを編集(https://thelinuxcluster.com/2020/05/13/compiling-scalapack-2-0-2-on-centos-7/)
# BLASLIB       = /usr/local/BLAS/libblas.a
# LAPACKLIB     = /usr/local/lapack/liblapack.a
cd
cp -r scalapack-2.0.2 /usr/local/
```

### 動作検証
```bash
cd
wget http://www.netlib.org/scalapack/examples/sample_pssyev_call.f
mpif90 -O3 -o TEST_sample_pssyev_call sample_pssyev_call.f /usr/local/scalapack-2.0.2/libscalapack.a -llapack -L/usr/local/lapack/lib -lblas -L/usr/local/BLAS
mpirun TEST_sample_pssyev_call
```

## FFTW3(v3.3.9)の導入
```bash
cd
wget http://www.fftw.org/fftw-3.3.9.tar.gz
tar -xvzof fftw-3.3.9.tar.gz
cd fftw-3.3.9
CC=gcc F77=gfortran CFLAGS="-O3 -fno-tree-vectorize -fexceptions" FFLAGS="-O3 -fno-tree-vectorize -fexceptions" ./configure --prefix=/usr/local --enable-threads --enable-shared --enable-static
make
sudo make install
sudo vi /etc/ld.so.conf
# /usr/local/lib を追記して保存
sudo /sbin/ldconfig
```
