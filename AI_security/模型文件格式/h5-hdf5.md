# HDF

​		层次数据格式，`Hierarchical Data Format`， 最开始由美国国家超算中心研发，后来由一个非盈利组织 `HDF Group`支持。`HDF` 支持多种商业及非商业的软件平台，包括 MATLAB、Java、Python、R 和 Julia 等，现在也提供 Spark，版本包括 HDF4 和现在使用广泛的 HDF5。`h5` 是 HDF5 文件格式的标准后缀（也有`.hdf5`的后缀），对于存储大量数据而言拥有极大的优势。

<font color="red">注：HDF5与早期的HDF4不兼容</font>

# h5/hdf5

​		`h5/hdf5` 使用类似”文件目录“的结构，允许以多种不同的结构化方式组织文件中的数据，就像处理计算机上的文件一样。同时，h5 还允许嵌入元数据，使其具有自描述性。

## 数据组织方式

​		h5 文件中主要包含两种结构：组`“group”`和数据集`“dataset”`，类似于文件结构中的目录和文件

​		- dataset：类似数组形式的数据集合，像 `numpy` 数组一样工作，一个 `dataset` 即一个 `numpy.ndarray`，具体的内容可以是图像、表格甚至是 pdf 文件。

​		- group：一种可嵌套结构，包含其他 `dataset` 或 `group`，像字典一样工作

​		元数据（metadata）主要通过属性和结构化数据本身两种方式存储，属性就是附着在组（group）、数据集（dataset）上的键值对，类似于文件系统中的扩展属性

![](./images/h5.png)

## 代码操作

​		在 python 中对 h5 文件的操作通过库 `h5py`，我们先写一个示例 h5 文件，结构如上图

```python
import h5py
import numpy as np

with h5py.File('../data/animal.h5', 'a') as f:
    # 在根目录下创建一个总览介绍动物种类的dataset，字符串应当字节化
    f.create_dataset('animal_included', data=np.array(['dogs'.encode(), 'cats'.encode()])) 
    # 在根目录下创建group文件夹：dogs
    dogs_group = f.create_group('dogs')
    # 根目录下有一个含5张猫图片的dataset文件
    f.create_dataset('cats', data=np.array(np.random.randn(5, 64, 64, 3)))
    # 在 dogs 文件夹下分别创建两个 dataset，一张哈士奇图片和一张柴犬的图片
    dogs_group.create_dataset("husky", data=np.random.randn(64, 64, 3))
    dogs_group.create_dataset("shiba", data=np.random.randn(64, 64, 3))
```

​		接着读取这个文件

```python
with h5py.File('../data/animal.h5', 'r') as f:
    for fkey in f.keys():
    	print(f'f[key]:{f[fkey]}\nfkey:{fkey}\nf[fkey].name:{f[fkey].name}\n---------------\n')

----------------------------------------------------------------
'''打印结果如下：
f[key]:<HDF5 dataset "animal_included": shape (2,), type "|S4">
fkey:animal_included
f[fkey].name:/animal_included
---------------

f[key]:<HDF5 dataset "cats": shape (5, 64, 64, 3), type "<f8">
fkey:cats
f[fkey].name:/cats
---------------

f[key]:<HDF5 group "/dogs" (2 members)>
fkey:dogs
f[fkey].name:/dogs
---------------
'''
----------------------------------------------------------------
```

​		文件对象`f`表示h5文件的根目录(root group)，又 `group` 是按字典的方式工作的，通过 `f.keys()` 来找到根目录下的所有 `dataset` 和 `group` 的 `key`，然后通过 `key` 来访问各个 `dataset` 或 `group` 对象。这些 `key-value` 就是 `h5` 文件中的元数据

结果解析：

```
1. 这个h5文件下有 1 个叫 dogs 的文件夹(group)和 2 个文件(dataset)，它们分别叫 animal_include, cats
	dogs group下有 2 个成员但我们不知道它是 group 还是 dataset

2.我们可以发现key和name的区别:   
	上层 group 对象是通过 key 来访问下层 dataset 或 group 的而不是通过 name 来访问的；
	因为 name 属性它是 dataset 或 group 的绝对路径并非是真正的 "name"，key 才是真正的"name"     
	绝对路径：比如访问 f['dogs']['husky'].name 得到：/dogs/husky，它表示根目录下有 dogs 这个挂载点，dogs 下又挂载了 husky
```

参考链接：https://blog.csdn.net/buchidanhuang/article/details/89716252

### 用途

​		一般在深度学习中用来存储训练数据或训练后的参数