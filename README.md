企业商品销售、配送与质量问题分析报告

项目概述

本项目基于某企业6种商品的销售及用户反馈数据，从配送服务、销售区域潜力、商品质量三个维度进行深入分析，
为优化供应链管理、区域市场策略、质量管控体系提供数据支持。

📊 数据洞察

核心结论
1. 配送服务优化  
   • 货品4→西北线路  

   • 货品2→马来西亚线路

   这两条线路存在显著时效问题，需紧急优化配送流程

2. 市场潜力区域  
   • 货品2在华东地区尚有显著市场空间，值得扩大投入  
 
   • 货品2在西北地区投入产出比失衡，建议减少资源分配

3. 商品质量问题  
   • 货品1、2、4的合格率较低  

   这三款商品需加强质量管控，建议扩大抽检比例至5%

---

🔍 分析过程

一、数据预处理

1. 数据清洗
• 重复值处理：检查并删除重复的订单记录（发现5条重复记录已处理）

• 缺失值管理：

  • 删除`数量`为空的记录（4条）、'订单号'和'货品交货状况'为空的记录（2条）、重复值9条
  
  • 订单行，对分析无关紧要，可以考虑删除

• 格式标准化：

  ```python
  # 时间格式统一
  data['销售时间'] = pd.to_datetime(data['销售时间'])

  
  # 取出销售金额列，对每一个数据进行清洗
  # 编写自定义过滤函数：删除逗号，转成float，如果是万元则*10000，否则，删除元
  def data_deal(number):
    if number.find('万元')!= -1:#找到带有万元的，取出数字，去掉逗号，转成float，*10000
        number_new = float(number[:number.find('万元')].replace(',',''))*10000
        pass
    else: #找到带有元的，删除元，删除逗号，转成float
        number_new = float(number.replace('元','').replace(',',''))
        pass
    return number_new
data['销售金额'] = data['销售金额'].map(data_deal)
  ```

2. 异常值检测
```python
# 1.销售金额为0的情况，删除
# 2.产生严重的数据左偏情况（电商领域的2/8法则很正常。）
data = data[data['销售金额']!=0]
```

3. 辅助指标生成
  ```python
  # 提取月份
  data['月份'] = data['销售时间'].apply(lambda x:x.month)
  
  # 按月计算按时交货率
  data['货品交货状况'] = data['货品交货状况'].str.strip()
  data1 = data.groupby(['月份','货品交货状况']).size().unstack()
  data1['按时交货率'] = data1['按时交货']/(data1['按时交货']+data1['晚交货'])
  ```

二、核心分析维度

1. 配送时效分析
```python
# 计算各商品各区域的按时交货率
data1 = data.groupby(['销售区域','货品交货状况']).size().unstack()
data1['按时交货率'] = data1['按时交货']/(data1['按时交货']+data1['晚交货'])
print(data1.sort_values(by='按时交货率',ascending=False))
#西北地区存在突出的延时交货问题，急需解決

# 可视化表现最差的前5条线路
worst_routes = data1.head(5)
worst_routes.plot(
    kind='barh', 
    figsize=(10, 6), 
    title='配送时效最差的5条线路',
    xlabel='平均配送天数',
    ylabel='区域-商品组合'
)
```

2. 区域市场分析
```python
# 计算不同月份和区域货品2的销售数量
data1 = data.groupby(['月份','销售区域','货品'])['数量'].sum().unstack()
data1['货品2']
#货品2在10，12月份销量猛增，原因主要发生在原有销售区域（华东）
#同样，分析出在7，8，9，11月份销售数量还有很大提升空间，可以适当加大营销力度


# 生成区域热力图
import seaborn as sns
plt.figure(figsize=(12, 8))
sns.heatmap(
    data1['货品2'].unstack(),
    annot=True, 
    cmap='YlOrRd',
    cbar_kws={'label': '货品2区域销售量'}
)
plt.title('区域市场表现热力图')
plt.show()
```

3. 商品质量问题分析
```python
# 计算质量问题指标
quality_issues = data.groupby('product').agg(
    return_rate=('returns', 'mean'),
    complaint_rate=('complaints', 'mean')
).sort_values('return_rate', ascending=False)

data['货品用户反馈'] = data['货品用户反馈'].str.strip()  #取出首位空格
data1 = data.groupby(['货品','销售区域'])['货品用户反馈'].value_counts().unstack()
# 确保所有反馈类型都存在（如果缺失则填充为0）
feedback_types = ['拒货', '返修', '质量合格']
for feedback in feedback_types:
    if feedback not in data1.columns:
        data1[feedback] = 0
data1['拒货率'] = data1['拒货'] /data1.sum(axis=1)  #按行进行求和汇总
data1['返修率'] = data1['返修'] /data1.sum(axis=1)
data1['合格率'] = data1['质量合格'] /data1.sum(axis=1)
data1.sort_values(['合格率','返修率','拒货率'],ascending=False)


# 质量问题分布饼图（仅展示前3款问题商品）
quality_issues = data1.sort_values('合格率', ascending=True).head(3)

# 确保 '货品' 是列名（如果之前是索引，需要重置索引）
if isinstance(quality_issues.index, pd.MultiIndex):
    quality_issues = quality_issues.reset_index()

# 检查 '货品' 列是否存在
if '货品' not in quality_issues.columns:
    # 如果 '货品' 是多级索引的一部分，提取它
    if len(quality_issues.index.names) > 0 and quality_issues.index.names[0] == '货品':
        quality_issues = quality_issues.reset_index(level='货品')

# 绘制饼图
plt.figure(figsize=(8, 8))
plt.pie(
    quality_issues['合格率'], 
    labels=quality_issues['货品'],
    autopct='%1.1f%%',
    startangle=90,
    colors=sns.color_palette('Reds', 3)
)
plt.title('质量问题产品分布')
plt.show()


🔧 技术实现

核心代码结构
```
analysis/
├── data/          # 原始数据与清洗后数据
├── notebooks/     # Jupyter分析笔记本
│   ├── 01_Data_Cleaning.ipynb
│   ├── 02_Delivery_Analysis.ipynb
│   └── 03_Quality_Assessment.ipynb
├── visuals/       # 生成的所有图表
└── requirements.txt   # 项目依赖
```

关键依赖库
```
pandas>=1.3.0
matplotlib>=3.4.0
seaborn>=0.11.0
numpy>=1.21.0
plotly>=5.0.0
```

📌 下一步建议
1. 针对TOP2问题线路进行配送流程优化试点
2. 在华东地区货品2市场开展专项营销活动
3. 建立货品1/2/4的高风险批次预警机制
4. 每季度更新质量抽检标准阈值

📋 项目设置

```bash
# 克隆仓库
git clone https://github.com/yourname/product-analysis.git

# 安装依赖
pip install -r requirements.txt

# 启动Jupyter Lab
jupyter lab
```


✏️ 分析说明
• 所有图表均基于企业2016年7-12月的运营数据

• 配送时效以"天"为单位，质量问题数据按退货比例计算

• 区域划分依据企业标准市场区域代码

• 销售单位：元


---

该报告从业务痛点出发，通过多维数据分析和可视化呈现，为企业提供了清晰的问题识别和决策支持框架。
