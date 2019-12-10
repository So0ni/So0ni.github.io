---
title: Ubuntu 18.04 x64 单机多核并行WRF和WPS编译安装
date: 2018-11-29 20:18:35
tags: meteorology
---
WRF全称Weather Research and Forecasting Model, 是一个天气研究与预报模型.可以用来进行精细尺度的天气模拟与预报。
WPS全称WRF Pre-processing System，即WRF预处理系统，用来为WRF模型准备输入数据；
如果只是做理想实验(idealized modeling)，就不需要用WPS处理真实数据。

除了WPS与WRF两大核心模块外，WRF系统还有很多附加模块：比如用于数据同化的WRF-DA，用于化学传输的WRF-chem，用于林火模拟的WRF-fire。

## 安装环境

* Ubuntu 18.04 x64 LTS
* Intel Core i5-4210M

<!-- more -->

### 依赖包

* gfortran
* zlib1g-dev
* libpng-dev 
* libjasper1
* libjasper-dev
* libjpeg-turbo
* nasm
* cmake
* build-essential
* m4
* hdf5-1.10.4
* netcdf-c-4.6.2
* netcdf-fortran-4.4.4
* openmpi-1.6.5

#### 主程序

* WRFV3.9.1.1
* WPSV3.9.1

## 开始安装

### 安装二进制依赖

```bash
sudo apt install -y build-essential gfortran zlib1g-dev m4 libpng-dev nasm cmake
```

18.04的默认软件库里没有libjasper-dev包，但是我们可以借用16.04的包，所以我们先添加16.04的源，然后继续安装。

```bash
sudo add-apt-repository "deb https://mirrors.tuna.tsinghua.edu.cn/ubuntu xenial-security main"
sudo apt update
sudo apt install libjasper1 libjasper-dev
```

### 编译安装其他依赖

#### hdf5-1.10.4

```bash
wget -c https://s3.amazonaws.com/hdf-wordpress-1/wp-content/uploads/manual/HDF5/HDF5_1_10_4/hdf5-1.10.4.tar.gz
tar -zxf hdf5-1.10.4.tar.gz
cd hdf5-1.10.4
./configure --prefix=/usr/local --enable-fortran --enable-cxx
make -j4
make check
sudo make install
sudo make check-install
cd ../
```

#### netcdf-c-4.6.2

```bash
wget -c https://www.unidata.ucar.edu/downloads/netcdf/ftp/netcdf-c-4.6.2.tar.gz
tar -zxf netcdf-c-4.6.2.tar.gz
cd netcdf-c-4.6.2
./configure --enable-netcdf-4 LDFLAGS="-L/usr/local/lib" CPPFLAGS="-I/usr/local/include"
make -j4
sudo make install
cd ../
```

#### netcdf-fortran-4.4.4

```bash
wget -c https://www.unidata.ucar.edu/downloads/netcdf/ftp/netcdf-fortran-4.4.4.tar.gz
tar -zxf netcdf-fortran-4.4.4.tar.gz
cd netcdf-fortran-4.4.4
./configure --prefix=/usr/local LDFLAGS="-L/usr/local/lib" CPPFLAGS="-I/usr/local/include" FC=gfortran
make -j4
sudo make install 
cd ../
```

netcdf从某一版本开始就拆分成了fortran和c版本，我们都需要安装。

#### libjpeg-turbo

```bash
git clone https://github.com/libjpeg-turbo/libjpeg-turbo.git
cd libjpeg-turbo
cmake -G"Unix Makefiles" -DWITH_JPEG8=1 -DCMAKE_INSTALL_LIBDIR:PATH=/usr/local/
make -j4
sudo make install
cd ../
```

libjpeg是一款开源的jpeg压缩解压库。libjpeg-turbo是它的升级版，性能有所提升.
libjpeg-turbo是libjpeg的一个分支，SIMD指令来加速JPEG编码和解码基础效率。
现在大部分项目使用libjpeg-turbo而不是libjpeg。

#### OpenMPI-1.6.5

```bash
wget -c https://download.open-mpi.org/release/open-mpi/v1.6/openmpi-1.6.5.tar.gz
tar -zxf openmpi-1.6.5.tar.gz
cd openmpi-1.6.5
./configure --prefix=/usr/local
make all -j4
sudo make install
cd ../
```

安装完所有依赖后，我们更新一下动态库链接。

```bash
sudo /sbin/ldconfig -v
```

---

### 编译主程序

#### WRFV3.9.1.1

```bash 
wget -c http://www2.mmm.ucar.edu/wrf/src/WRFV3.9.1.1.TAR.gz
tar -zxf WRFV3.9.1.1.TAR.gz
cd WRFV3
./configure
./compile wrf
./compile em_real
cd ../
```

编译前配置时，会出现很多编译选项供选择。每一个选项的前半部分通常是在描述编译器与运行环境，根据计算机实际情况选择即可。
后半部分是并行选项： 

* serial  串行计算。
* smpar  内存共享并行计算(shared memory option)，即使用openMP，大部分多核电脑都支持这项功能。
* dmpar  分布式并行计算(distributed memory option)，即使用MPI 进行并行计算，主要用在计算集群，单个电脑就没必要用了。
* dm+sm  同时使用openMP与MPI两种并行方式。

由于我们目标是单机多核并行计算，且我们是x64 Ubuntu使用gfortran编译器，所以我们选择`33`。

选择完编译选项后，会出现提示选择嵌套选项，一般就选 basic。

* 1=basic, 有嵌套
* 2=preset moves, 指定移动网格
* 3=vortex following 自动移动网格


如果编译正常，在main目录下会看到

* ndown.exe
* nup.exe
* real.exe
* tc.exe
* wrf.exe

这5个可执行程序

如果遇到 (反正我是没遇到)

```bash
start_em.f90:209.60:ALLOCATE( clat_glob(ids:ide,jds:jde), STAT=alloc_status, ERRMSG=alloc_err_1 Error: Syntax error in DEALLOCATE statement at (1)1234start_em.f90:209.60: ALLOCATE( clat_glob(ids:ide,jds:jde), STAT=alloc_status, ERRMSG=alloc_err_ 1Error: Syntax error in DEALLOCATE statement at (1)
```

可能是因为使用的fortran编译器不支持ALLOCATE函数的ERRMSG参数，修改源代码文件`start_em.F`中的代码，去掉

```
, ERRMSG=alloc_err_message
```

然后重新编译。

#### WPSV3.9.1

```bash 
wget -c http://www2.mmm.ucar.edu/wrf/src/WPSV3.9.1.TAR.gz
tar -zxf WPSV3.9.1.TAR.gz
cd WPS
./configure
sed -i "s/-lnetcdff -lnetcdf/-lnetcdff -lnetcdf -lgomp/g" configure.wps
./compile
cd ../
```

__注意：WPS编译时会在相同目录下寻找已经编译好的WRF目录，也就是说，源代码目录 WPS 要跟 WRFV3 放在同一个父目录下__

由于WPS只有多机并行和单机串行方式，所以编译选项选择时我们选择`1` 单机串行。

`sed -i "s/-lnetcdff -lnetcdf/-lnetcdff -lnetcdf -lgomp/g" configure.wps`

目的是为了在`-lnetcdff -lnetcdf`后面添加`-lgomp`， 这个能解决OpenMP报错问题。

```bash 
undefined reference to `GOMP_parallel' ...
```

编译成功后，最后会在WPS代码根目录得到以下三个程序链接

* geogrid.exe -> geogrid/src/geogrid.exe
* ungrib.exe -> ungrib/src/ungrib.exe 
* metgrid.exe -> metgrid/src/metgrid.exe

到此我们就成功编译了WRF和WPS。

注意： WPS从V3.8开始，将GEOG地形数据场插值数据源换成了
`30-arc-second USGS GMTED2010` `MODIS FPAR` `21-class MODIS`。
如果需要使用V3.7以及之前的地形文件数据，需要把`namelist.wps`文件中的
`geog_data_res`选项设置为` = 'gtopo_10m+usgs_10m+nesdis_greenfrac+10m','gtopo_2m+usgs_2m+nesdis_greenfrac+2m',`。
之前版本的`geog_data_res='10m','2m',`不再可用。

