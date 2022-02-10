## Similarity

##### Cosine Similarity

公式：

![image-20220119144038444](https://tva1.sinaimg.cn/large/008i3skNgy1gyiz61qsq3j30qo07ut95.jpg)

代码：

```java
double dotProduct = 0.0;
double normA = 0.0;
double normB = 0.0;
for (int i = 0; i < vectorA.length; i++) {
  dotProduct += vectorA[i] * vectorB[i];
  normA += Math.pow(vectorA[i], 2);
  normB += Math.pow(vectorB[i], 2);
}
double d = dotProduct / (Math.sqrt(normA) * Math.sqrt(normB));
return Precision.round(d, 8);
```

#### 欧氏距离

![](https://tva1.sinaimg.cn/large/008i3skNgy1gyj0xdp7rzj3110074mxj.jpg)

#### 曼哈顿距离

![](https://tva1.sinaimg.cn/large/008i3skNgy1gyj11ir4boj30h004sglk.jpg)