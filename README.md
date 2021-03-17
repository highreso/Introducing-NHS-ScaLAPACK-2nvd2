# nvd2系インスタンスにNvidia HPC SDKと数値計算ライブラリ群を導入する
## 目的
1. OpenACCの有効化(GPU)
2. FTTW3の動作(CPU)
3. ScaLAPACKの動作(CPU)

## 実行上の注意
ツールの導入時は必ずホームディレクトリ(`/home/user/`)内ににて行うようにします。<br/>
`cd`と打てばホームディレクトリに移動できるので、ツールをインストールする際は必ずこの操作を行うようにしてください。<br/>
基本的には、以下のコマンドを順番に実行すれば、全てのツールがインストールできるようになっておりますが、一部ご使用状態に合わせてファイルの編集を行う必要がありますのであらかじめご注意のほどお願いいたします。

## 事前準備
- 会員登録
- `nvd2dl`インスタンスの作成
- インスタンスへの接続確認
[参考](https://gpu-advance.highreso.jp/blog/?p=232)
- VSCodeからのファイル編集環境確立(オプション)
[参考](https://gpu-advance.highreso.jp/blog/?p=301)

## makeとvimの導入
```bash
sudo apt update
sudo apt install -y make vim
```

## Nvidi HPC SDKの導入
```bash
cd
wget https://developer.download.nvidia.com/hpc-sdk/20.11/nvhpc-20-11_20.11_amd64.deb
wget https://developer.download.nvidia.com/hpc-sdk/20.11/nvhpc-2020_20.11_amd64.deb 
wget https://developer.download.nvidia.com/hpc-sdk/20.11/nvhpc-20-11-cuda-multi_20.11_amd64.deb
sudo apt-get install ./nvhpc-20-11_20.11_amd64.deb ./nvhpc-2020_20.11_amd64.deb ./nvhpc-20-11-cuda-multi_20.11_amd64.deb
```

### パスの追加
Nvidia HPC SDKのインストールが完了したら、「パスを通す」という作業を行います。<br/>
コマンド(コンパイラで言えばコンパイル時に頭につけるgfortran, pgfortran, mpifortなど)を有効化するために必要です（厳密には、フルパスで実行せずに済ますための処理です）。<br/>
下記のコマンドを実行することでパスの追加は完了します。
```bash
export PATH="$PATH:/opt/nvidia/hpc_sdk/Linux_x86_64/20.11/compilers/bin"
```

注意: 上記のコマンド実行の場合、インスタンスにターミナルからアクセスしている時しか有効化されません。
毎回上記のコマンドを打ちたくない場合は以下の節で設定を行う必要があります。

### パスの追加（発展編）
前節でのパスの通し方では、インスタンスからログアウトして再度ログインした際、パスがなくなってしまいます。<br/>
これを回避するため、「インスタンスンのログインをトリガとして上記のコマンドを実行する」処置を取ります。<br/>
具体的には、`.bashrc`というファイルを編集します。<br/>
```bash
cd
code .bashrc
## これで開いた「.bashrc」というファイルの一番下に以下の文を追加して保存
export PATH="$PATH:/opt/nvidia/hpc_sdk/Linux_x86_64/20.11/compilers/bin"

## 追記と保存が完了したら下記コマンドを実行
source .bashrc
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
cd lapack-3.4.0
cp INSTALL/make.inc.gfortran make.inc
## make.incをhttps://thelinuxcluster.com/2012/04/09/building-lapack-3-4-with-intel-and-gnu-compiler/ に従って編集します
# 下記のように該当変数の書き換え・追加を行います
# PLAT = _LINUX
# OPTS = -O2 -m64 -fPIC
# NOOPT = -m64 -fPIC

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
## SLmak.inceを編集し下記の環境変数を設定します(https://thelinuxcluster.com/2020/05/13/compiling-scalapack-2-0-2-on-centos-7/)
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
# または mpifort -O3 -o TEST_sample_pssyev_call sample_pssyev_call.f /usr/local/scalapack-2.0.2/libscalapack.a -llapack -L/usr/local/lapack/lib -lblas -L/usr/local/BLAS
mpirun ./TEST_sample_pssyev_call
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

### 動作検証
以下のコードを用いて動作検証を行う
```Fortran
program sample
  implicit none
  include 'fftw3.f'

  write(*, *) FFTW_ESTIMATE

  stop
end program sample
```
`sample.f90`というファイル名で保存し、FFTW3を用いてコンパイルを行う：
```bash
mpif90 -c sample.f90 -I/usr/local/include
mpif90 -o fftw-test sample.o -L/usr/local/lib -lfftw3
./fftw-test  # 64という値が出力されれば動作確認成功
```