---
layout: post
title: "PPG-DISTILL: Efficient Photoplethysmography Signals Analysis via Foundation Model Distillation"
description: 
categories: "论文"
---
# 前置知识
## KD
#### 全局知识蒸馏
![alt text](/images/posts/论文项目/{94027F49-0E8B-4608-8B71-5B46B8806549}.png)
![alt text](/images/posts/论文项目/{18C3B46B-47FE-4C78-B234-D773B14BF049}.png)
![alt text](/images/posts/论文项目/{2EC8914E-9F6A-4AAA-A285-8C65AAF80E3D}.png)
![alt text](/images/posts/论文项目/{F38BE548-9755-412A-9729-2F4CF03E0E11}.png)
![alt text](/images/posts/论文项目/{CB52B483-11C7-434D-AA76-7DF77F732C0B}.png)
#### why软标签比硬标签要好
## 特征蒸馏与匹配
#### 特征就是模型在计算过程中产生的中间表示
#### 匹配：最小化两个特征向量之间的距离



---------------------
## 方法：
#### 在传统的全局蒸馏基础上，增加了两个局部蒸馏的损失函数：形态蒸馏和节律蒸馏
![alt text](/images/posts/论文项目/{EEB222DF-849F-4DD0-868E-8B5ACC827410}.png)
 - **形态蒸馏**
    适配器 (Adapter) ：因为教师（大模型）和学生（小模型）提取的特征维度可能不一样，所以用一个线性层把教师的特征维度 d_t 映射到学生的维度 d_s，这样才能比较
    InfoNCE（对比学习损失）
    ![alt text](/images/posts/论文项目/{4D594F16-329D-42BC-AB61-D3E4FC3950BB}.png)
    正样本：Zii   
    负样本：Zij
    -log操作最大化正样本对的占比
- **节律蒸馏**
    不比较学生和老师的特征，而是比较特征间的关系————**关系蒸馏**
    >![alt text](/images/posts/论文项目/{55C54E5F-887D-4F8F-BFD7-8D39EE363E77}.png)
    >解析：![alt text](/images/posts/论文项目/{85189FB1-CF83-4A26-AE7F-CEBDEF9AD4ED}.png)


 - 联合优化
![alt text](/images/posts/论文项目/{F5BAE97D-F26D-4AFE-ACC8-EF0244247C61}.png)
#### 损失
![alt text](/images/posts/论文项目/{5FB7D6D9-F22B-42DD-B904-2139DA9F4985}.png)









*你可以（也应该）在你完成CM-SSL的预训练后，直接应用PPG-DISTILL的框架来压缩你的模型，以实现“系统构建”和“高效推理”的目标*



---------------------
## 新的研究视角或思路：
它提供了一个关键视角：不要只匹配全局，要深入到Patch级别去匹配局部结构。

#### 它最巧妙的思路是：跨领域借鉴。它将对比学习 (Contrastive Learning) 的思想（InfoNCE Loss）和关系型KD（knowledge distill） (Relational KD)  的思想，创造性地用于解决PPG的局部信息保留问题。

这不再是预训练的“下一步”，而是从“科研”走向“工程/系统”的第一步

