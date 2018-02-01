---
title: 拖动排序后台逻辑思考过程
tags:
  - sort
categories: QA
permalink: drag-order-thinking
date: 2018-01-27 11:08:12
---

## 需求
实现类似[这个](http://www.jq22.com/yanshi1680)的排序后台逻辑

## 思考
- 必要字段主键`id`，`sequence`
- 前端传递最后的值直接保存?
  - 如果这个不是管理员使用的话，可能被注入错乱的相同排序序号，就无法正确的排序了。
  - 时间比较紧急，前端需要去查找插件的使用方法等。
  - 通常前端是实现页面效果和简单的控制，复杂的逻辑还是后端实现。
- 排序逻辑思考
  - 最简单的情况 <br/>
    初始值如下
    ```
    id   -   sequence
    A    -      1
    B    -      2
    C    -      3
    D    -      4
    ```
    现在将`D-4`和`C-3`调换位置
    ```
    id   -   sequence
    A    -      1
    B    -      2
    D    -      4
    C    -      3
    ```
    想要保存这个结果必须修正`C`和`D`的`sequence` <br/>
    结果如下
    ```
    id   -   sequence
    A    -      1
    B    -      2
    D    -      3
    C    -      4
    ```
    那么这个过程中只是调换了`C`和`D`的`sequence`
   - 最复杂的情况如下
     初始值如下
     ```
     id   -   sequence
     A    -      1
     B    -      2
     C    -      3
     D    -      4
     ```
     现在将`D-4`拖动到顶层
     ```
     id   -   sequence
     D    -      4
     A    -      1
     B    -      2
     C    -      3
     ```
     想要保存这个结果必须修正所有的`sequence`
     结果如下
     ```
     id   -   sequence
     D    -      1
     A    -      2
     B    -      3
     C    -      4
     ```
  - 小结：这个排序其实就是把`sequence`重新分配，只要将所有的`sequence`重新排序，然后按照顺序分配即可。
## 解决方案  
  - 前端传递id数据库自关联修改 `sequence`
    - 问题：数据库不会简单的根据传递的参数来排序
      - 传递参数 `D,A,B,C` <br/>
        期望:
        ```
        id   -   sequence
        D    -      4
        A    -      1
        B    -      2
        C    -      3
        ```
        实际情况：
        ```
        id   -   sequence
        A    -      1
        B    -      2
        C    -      3
        D    -      4
        ```
        通过查询查询资料这个需要写复杂的函数来解决，sql中不宜写复杂的函数，难以维护。
  - 在`java`中排序然后批量更新(逐条更新会有并发问题的可能性)。
   
      ```java
      public BaseResp<Void> sort(List<DocumentSort> sortList) {
       List<Integer> sequenceList = new ArrayList<>(sortList.size());
          sortList.forEach(d -> sequenceList.add(d.getSequence()));
          Collections.sort(sequenceList);
          List<DocumentCatalog> needUpdate = new LinkedList<>();
          for (int i = 0; i < sortList.size(); i++) {
              DocumentSort document = sortList.get(i);
              if(Objects.equals(document.getSequence(), sequenceList.get(i))){
                  continue;
              }
              DocumentCatalog catalog = new DocumentCatalog();
              catalog.setId(document.getId());
              catalog.setSequence(sequenceList.get(i));
              needUpdate.add(catalog);
          }
          if(0 != needUpdate.size()) {
              documentCatalogMapper.batchUpdate(needUpdate);
      }
      ```
     - 通过测试这段代码还是存在问题：
       - 当前端拖动的速度过快，数据`sequence`重复了！！！
         - java并发问题?
           - 这个方法没有共享变量
           - `Collections.sort(sequenceList)`调用的是自身的排序，排查并发问题
           - 这个是批量更新
         - 核查了前端传递的数据，不存在传递相同的`sequence`
         - 偶然发生`debug`不合适
       - 详细日志，重现问题
         1. 初始值
         ```
         id   -   sequence  db-sequence
         A    -      1           1
         B    -      2           2
         C    -      3           3
         D    -      4           4
         ```
         
         2. 将`D`移至第一个
         ```
         id   -   sequence  db-sequence
         D    -      1           1
         A    -      2           2
         B    -      3           3
         C    -      4           4
         ``` 
               
         3. 将`A`和`B`调换  此时前端传递的`sequence`没变
         ```
         id   -   sequence  db-sequence
         D    -      4           1
         B    -      2           3
         A    -      1           2
         C    -      3           4
         ```
         
         4. 重排后结果：
         ```
         id   -   sequence  db-sequence
         D    -      1           1
         B    -      2           3
         A    -      3           2
         C    -      4           4
         ```          
         
         5. 增量更新就不会更新`B`的`sequence`
         最后结果数据库
         ```
         id   -   sequence  db-sequence
         D    -      1           1
         B    -      2           3
         A    -      3           3
         C    -      4           4
         ```          
         
         6. 此时出现了2个`3`
  - 最终代码全量更新得到正确结果：       
      ```java
      public BaseResp<Void> sort(List<DocumentSort> sortList) {
       List<Integer> sequenceList = new ArrayList<>(sortList.size());
          sortList.forEach(d -> sequenceList.add(d.getSequence()));
          Collections.sort(sequenceList);
          List<DocumentCatalog> needUpdate = new LinkedList<>();
          for (int i = 0; i < sortList.size(); i++) {
              DocumentSort document = sortList.get(i);
              DocumentCatalog catalog = new DocumentCatalog();
              catalog.setId(document.getId());
              catalog.setSequence(sequenceList.get(i));
              needUpdate.add(catalog);
          }
          if(0 != needUpdate.size()) {
              documentCatalogMapper.batchUpdate(needUpdate);
      }
      ```  
