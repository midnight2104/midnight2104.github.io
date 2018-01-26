- 标准化：
> 处理数据时，要知道数据本身是怎样分布的，是需要截断处理还是缩减范围？截断会丢失数据，缩减范围会增加各个数据之间的对比差异。所以根据数据本身情况而定，不要想当然的随机选择一种方式
```py
#方式 1
X_ = (X_ - np.mean(X_, axis=0)) / np.std(X_, axis=0)

#方式 2
x = (x - np.min(x)) / np.max(x)

#方式 3
img(img < 0) = 0;
img(img > 255) = 255;

```

- 独立热编码
```py
#方式1
def one_hot_encode(x):
    targets = np.array(x).reshape(-1)
    return np.eye(10)[targets]

#方式2
from sklearn.preprocessing import LabelBinarizer 

def one_hot_encode(x):
    label_binarizer = LabelBinarizer() 
    label_binarizer.fit(range(10))
    return label_binarizer.transform(x)
```

- 随机散列数据
```py
from sklearn.model_selection import StratifiedShuffleSplit
ss = StratifiedShuffleSplit(n_splits=1, test_size=0.2)

train_idx,val_idx = next(ss.split(codes, labels))

half_val = int(len(val_idx)/2)
val_idx,test_idx = val_idx[:half_val],val_idx[half_val:]

train_x, train_y = codes[train_idx],labels_vecs[train_idx]
val_x, val_y = codes[val_idx],labels_vecs[val_idx]
test_x, test_y =  codes[test_idx],labels_vecs[test_idx]

```

- 权重的初始化，我们希望初始化的权重非常接近零但又不为零。一般建议在[-y,y]    ``$ y=1/\sqrt{n} $``(``$ n $`` 是输入的数量).

- bias的初始化，建议使用tf.zeros来初始化为零值
- 池化层放在激活层前面，可以减少激活层的计算量
- 尝试增加卷积层和全链接层，增强下神经网络的计算能力
- 卷积的输出通道数可以设为如 32 - 64 - 128 这样逐渐递增的形式；全连接层的节点数可以考虑类似 512 - 256 -128 的设定
- 卷积核设置上经验上建议设为奇数
