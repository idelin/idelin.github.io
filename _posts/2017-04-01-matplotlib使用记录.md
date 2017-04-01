# matplotlib使用记录

## 1.Linux服务器没有GUI的情况下使用matplotlib绘图
```python
import matplotlib as mpl
mpl.use('Agg')
```
必须添加在import matplotlib.pyplot之前


## 2.解决matplotlib中文乱码
```python
plt.rcParams['font.sans-serif'] = ['SimSun', 'SimHei', 'SimKai', 'PingFang SC']  
plt.rcParams['font.serif'] = ['SimSun', 'SimHei', 'SimKai', 'PingFang SC'] 
plt.rcParams['font.family'] = 'sans-serif'
```

## 3.解决matplotlib保存图像是负号'-'显示为方块的问题
```python
plt.rcParams['axes.unicode_minus'] = False
```

## 4. matplotlib保存图片时不截取图表内容
```python
plt.savefig("xxx.png", format='png', dpi=100, bbox_inches='tight')
```
> bbox_inches='tight'可控制所画图表被完整保存下来，而不被截取掉。

使用前
![](http://ongpr9izg.bkt.clouddn.com/2017-04-01-024436.jpg)
使用后
![](http://ongpr9izg.bkt.clouddn.com/2017-04-01-024349.jpg)

## 5.matplotlib控制图例位置
> plt.legend(bbox_to_anchor=(0.9, 0.9), fontsize="x-small")
 
## 6.控制图像边缘留白量
> plt.subplots_adjust(top=0.9, right=0.85)