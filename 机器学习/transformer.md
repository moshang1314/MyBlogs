![image-20221024221007846](https://cdn.jsdelivr.net/gh/moshang1314/myBlog/image/image-20221024221007846.png)

# 1 self-attention

1) 

> ![image-20221024221250559](https://cdn.jsdelivr.net/gh/moshang1314/myBlog/image/image-20221024221250559.png)

2) 

> ![image-20221024221430823](https://cdn.jsdelivr.net/gh/moshang1314/myBlog/image/image-20221024221430823.png)

3) 

> ![image-20221024221754827](https://cdn.jsdelivr.net/gh/moshang1314/myBlog/image/image-20221024221754827.png)

4) 计算向量间关联性

   > 1) ![](https://cdn.jsdelivr.net/gh/moshang1314/myBlog/image/image-20221024222019189.png)
   > 2) ![](https://cdn.jsdelivr.net/gh/moshang1314/myBlog/image/image-20221024222551905.png)
   > 3) ![image-20221024222746870](https://cdn.jsdelivr.net/gh/moshang1314/myBlog/image/image-20221024222746870.png)
   > 4) ![image-20221024222930408](https://cdn.jsdelivr.net/gh/moshang1314/myBlog/image/image-20221024222930408.png)
   > 5)  ![image-20221024223146363](E:/%E6%9C%BA%E5%99%A8%E5%AD%A6%E4%B9%A0/img/image-20221024223146363.png)
   
   5) 矩阵化
   
      > 1) ![image-20221024223529093](https://cdn.jsdelivr.net/gh/moshang1314/myBlog/image/image-20221024223529093.png)
      > 2)   ![image-20221024223849896](https://cdn.jsdelivr.net/gh/moshang1314/myBlog/image/image-20221024223849896.png)
      > 3) ![image-20221024224011522](https://cdn.jsdelivr.net/gh/moshang1314/myBlog/image/image-20221024224011522.png)
      > 4) ![image-20221024224331237](https://cdn.jsdelivr.net/gh/moshang1314/myBlog/image/image-20221024224331237.png)

# 2 Multi-head Self-attention

>1) ![image-20221024224630845](https://cdn.jsdelivr.net/gh/moshang1314/myBlog/image/image-20221024224630845.png)
>2) ![image-20221024224704675](https://cdn.jsdelivr.net/gh/moshang1314/myBlog/image/image-20221024224704675.png)
>3) ![image-20221024224735380](https://cdn.jsdelivr.net/gh/moshang1314/myBlog/image/image-20221024224735380.png)

# 3 Positional Ecoding

> 1) ![image-20221024225225542](https://cdn.jsdelivr.net/gh/moshang1314/myBlog/image/image-20221024225225542.png)
> 2) ![image-20221024225323702](https://cdn.jsdelivr.net/gh/moshang1314/myBlog/image/image-20221024225323702.png)

# 3 VIT

![image-20230107155240133](https://cdn.jsdelivr.net/gh/moshang1314/myBlog/image/image-20230107155240133.png)

![image-20230107165343893](https://cdn.jsdelivr.net/gh/moshang1314/myBlog/image/image-20230107165343893.png)

![image-20230107165403034](image-20230107165403034.png)

![image-20230107175014053](https://cdn.jsdelivr.net/gh/moshang1314/myBlog/image/image-20230107175014053.png)

![image-20230108115526836](https://cdn.jsdelivr.net/gh/moshang1314/myBlog/image/image-20230108115526836.png)

![image-20230108123020807](https://cdn.jsdelivr.net/gh/moshang1314/myBlog/image/image-20230108123020807.png)

![image-20230108183845622](https://cdn.jsdelivr.net/gh/moshang1314/myBlog/image/image-20230108183845622.png)

## 3. 1 Norm

![image-20230210204930175](https://cdn.jsdelivr.net/gh/moshang1314/myBlog/image/image-20230210204930175.png)

![image-20230210204942342](https://cdn.jsdelivr.net/gh/moshang1314/myBlog/image/image-20230210204942342.png)

![image-20230210205340120](https://cdn.jsdelivr.net/gh/moshang1314/myBlog/image/image-20230210205340120.png)

![image-20230212181217413](https://cdn.jsdelivr.net/gh/moshang1314/myBlog/image/image-20230212181217413.png)

# 4 Swin Transformer

## 4.1 模型流程

![image-20230222163157486](https://cdn.jsdelivr.net/gh/moshang1314/myBlog@main/image/image-20230222163157486.png)

![image-20230222195257652](https://cdn.jsdelivr.net/gh/moshang1314/myBlog@main/image/image-20230222195257652.png)

  ## 4.2 Patch Embedding

![](https://cdn.jsdelivr.net/gh/moshang1314/myBlog@main/image/image-20230222201245123.png)

![image-20230222213310973](https://cdn.jsdelivr.net/gh/moshang1314/myBlog@main/image/image-20230222213310973.png)

## 4.3 Windows Partition

![image-20230222201442449](https://cdn.jsdelivr.net/gh/moshang1314/myBlog@main/image/image-20230222201442449.png)

![image-20230223213722111](https://cdn.jsdelivr.net/gh/moshang1314/myBlog@main/image/image-20230223213722111.png)

## 4.4 Windows Muti-Head Self Attention

![image-20230222212114205](https://cdn.jsdelivr.net/gh/moshang1314/myBlog@main/image/image-20230222212114205.png)

## 4.5 Patch Merging

![image-20230222212247652](https://cdn.jsdelivr.net/gh/moshang1314/myBlog@main/image/image-20230222212247652.png)

![image-20230222224259482](https://cdn.jsdelivr.net/gh/moshang1314/myBlog@main/image/image-20230222224259482.png)

![image-20230223203744660](https://cdn.jsdelivr.net/gh/moshang1314/myBlog@main/image/image-20230223203744660.png)

## 4.6 Next Stage

![image-20230222212447883](https://cdn.jsdelivr.net/gh/moshang1314/myBlog@main/image/image-20230222212447883.png)

![image-20230222212636238](https://cdn.jsdelivr.net/gh/moshang1314/myBlog@main/image/image-20230222212636238.png)

## 4.7 Swin transformer:两个连续的Swin Block

> **W-MSA(Window Multi-head Self Attention)**
>
> **SW-MSA(Shifted Window Multi-head Self Attention)**

![image-20230222213029141](https://cdn.jsdelivr.net/gh/moshang1314/myBlog@main/image/image-20230222213029141.png)

### 4.7.1 W-MSA

![image-20230222221120630](https://cdn.jsdelivr.net/gh/moshang1314/myBlog@main/image/image-20230222221120630.png)

![image-20230224103827049](https://cdn.jsdelivr.net/gh/moshang1314/myBlog@main/image/image-20230224103827049.png)

#### 4.7.1.1 复杂度分析

> hw = patch_num

![image-20230222223527817](https://cdn.jsdelivr.net/gh/moshang1314/myBlog@main/image/image-20230222223527817.png)

![image-20230222223500959](https://cdn.jsdelivr.net/gh/moshang1314/myBlog@main/image/image-20230222223500959.png)

![image-20230222223827780](https://cdn.jsdelivr.net/gh/moshang1314/myBlog@main/image/image-20230222223827780.png)

### 4.7.2 SW-MSA

![image-20230224110807059](https://cdn.jsdelivr.net/gh/moshang1314/myBlog@main/image/image-20230224110807059.png)

![image-20230224111257396](https://cdn.jsdelivr.net/gh/moshang1314/myBlog@main/image/image-20230224111257396.png)

![image-20230224111636161](https://cdn.jsdelivr.net/gh/moshang1314/myBlog@main/image/image-20230224111636161.png)

> shift_size = window_size // 2
>
> x = x.roll(shifts=(-shift_size, -shift_size), axis=(1,2))
>
> ![image-20230224112050218](https://cdn.jsdelivr.net/gh/moshang1314/myBlog@main/image/image-20230224112050218.png)

![image-20230225103403121](https://cdn.jsdelivr.net/gh/moshang1314/myBlog@main/image/image-20230225103403121.png)

> typename _Operation::result_type std::binder1st<_Operation>::operator()(typename _Operation::second_argument_type&) const [with _Operation = times<int>; typename _Operation::result_type = int; typename _Operation::second_argument_type = const int]’ cannot be overloaded
>
> ​    operator()(typename _Operation::second_argument_type& __x) const
>
> operator()(const int& x) const;
>
> operator()(const int& x) const;
>
> (const times<int>)(const int&, int&);
>
> 
