# 任务介绍

知识库问答（knowledge base question answering,KB-QA）即给定自然语言问题，通过对问题进行语义理解和解析，进而利用知识库进行查询、推理得出答案。

# 数据格式介绍

每条数据由4行组成，第一行为问题，第二行为提取好的三元组[实体，属性，答案]，第三行为答案：

![image](https://user-images.githubusercontent.com/48178838/178094278-f592f1a2-d031-46fe-9763-e93e09ca1b2a.png)

# 模型选择

BertForTokenClassification + crf
BertForSequenceClassification

# 流程

## 构建数据库

使用的是mysql数据库，数据库命名为KB_QA，表名为nlpccqa

![image](https://user-images.githubusercontent.com/48178838/178094302-03ec5a1b-7f2d-4e43-9af3-e33ea558eb4b.png)

![image](https://user-images.githubusercontent.com/48178838/178094306-056d6eac-e538-4f4d-aa40-9286f81e448d.png)

表的内容为：

![image](https://user-images.githubusercontent.com/48178838/178094311-74d4c8cd-a243-43aa-8008-caa88e5a14fd.png)


## BertForTokenClassification+CRF

首先使用BertForTokenClassification分类器对问题进行命名实体识别序列标注得到实体，并通过CRF层学习潜在约束。

  - 构建NER数据集
  
  以下面这条数据为例：

  ![image](https://user-images.githubusercontent.com/48178838/178094332-7322f867-7906-4518-9deb-a0003678b719.png)

  标注后的结果为：
  ['O', 'B-LOC', 'I-LOC', 'I-LOC', 'I-LOC', 'O', 'O', 'O', 'O', 'O', 'O', 'O', 'O', 'O', 'O', 'O']
  对应：
  ['《','高','等','数','学','》','是','哪','个','出','版','社','出','版','的','？']

  ![image](https://user-images.githubusercontent.com/48178838/178094368-3d3edda1-f4e5-41a3-9f7a-e512a222f6d1.png)
  
  - BertForTokenClassification+CRF训练
  
    - 具体操作
    
    使用BertTokenizer.encode_plus()函数得到[input_ids,token_type_ids]，并在此基础上得到[input_ids,attention_mask,token_type_ids]。
    labels_ids通过NER数据集得到。
    最终形成每一条数据的input为：[input_ids,attention_mask,token_type_ids,labels_ids]
    
    - 模型
    
    因为BertForTokenClassification得到的是概率：

    ![image](https://user-images.githubusercontent.com/48178838/178094381-6bd83b3f-1803-4590-9137-390821a98b61.png)

    所以使用CRF来学习一些潜在的约束规则：

    ![image](https://user-images.githubusercontent.com/48178838/178094391-626ff41d-59f6-446e-b31f-464cf3d00e00.png)

    - 结果
    
    训练：
    
    ![image](https://user-images.githubusercontent.com/48178838/178094421-1f5c99a5-07a8-472c-bfe7-71dc56f95f37.png)

    测试：

    ![image](https://user-images.githubusercontent.com/48178838/178094426-211a5dca-0d2e-47a5-b119-622d0dc8dd02.png)

## BertForSequenceClassification

对于一个问题，如果前一步得到的实体存在于数据库中，且数据库中的属性也存在于问题中，那即得到了回答，但属性可能和问题中的提法不一致，所以使用BertForSequenceClassification进行相似判断

  - 构建SIM数据集
  
    - 具体操作
    
    每一条数据都找五个不同的属性当做负例。
    还是以上文的高等数学为例：
    
    ![image](https://user-images.githubusercontent.com/48178838/178094485-ccb5af2f-ee3f-4322-b70d-2d6e7e739cb6.png)


    - 结果
    
    每一个正例，有五个负例。
    
    ![image](https://user-images.githubusercontent.com/48178838/178094495-0bcac0cc-6264-4ee3-9452-177c842905fc.png)

  - BertForSequenceClassification训练
  
  ![image](https://user-images.githubusercontent.com/48178838/178094528-2ea6cfac-b4c1-4858-9ad7-475c4c1dfd6a.png)

  最终结果
  
  ![image](https://user-images.githubusercontent.com/48178838/178094532-34c18f27-eb87-4aa7-867b-89abebaa6642.png)

## 实验结果

（1）对于数据库有的实体且属性存在于问题中，直接从数据库得到答案：

![image](https://user-images.githubusercontent.com/48178838/178094569-e757e342-fb5d-4c65-92c5-08a2733195f0.png)

（2）对于数据库有的实体，属性不一致：

![image](https://user-images.githubusercontent.com/48178838/178094575-28d9f5bf-c157-4fab-ae94-c332664852d7.png)

（3）对于数据库中不存在的实体：

![image](https://user-images.githubusercontent.com/48178838/178094579-cf8cfca3-fe0b-4272-ac24-e8f37c63ed9b.png)


