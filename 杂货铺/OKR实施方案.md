# OKR实施方案

## 团队文化/价值观

1. 集体责任感

    团队共同承担项目成功或失败的责任，每个成员对特定的重要结果负责。
    
2. 敢于挑战

    直面问题、解决问题。
    
3. 可量化的成果

    以结果为导向。
    
## 落地方案

1. 周期

    以周为单位迭代，例行周会使用OKR进行工作汇报及更新。
    
2. 目标/关键结果数量

    每个人可设置3-5个目标（O），每个目标可与若干个关键结果（KR）相对应。关键结果的数目原则上不限制，可自行根据需求设置，但不建议太多。
    
3. 目标类型

    * 承诺型
        团队整体规划的事项，必须完成。
    &nbsp;
    * 日常型
        业务或临时需求。
    &nbsp; 
    * 愿景型
        自我挑战的事项，涉猎范围原则上不限制，最多可占用工作时长的20%，如：每周1天。

4. 关键结果权重

    每一个关键结果需要设置相应的权重值，范围：0.1-1.5。权重值的选取建议参考以下4个区间：
    * <= 0.6：日常型目标；
    * (0.6, 0.8]：承诺型一般目标；
    * (0.8, 1.0]：承诺型核心目标，技术或服务能力参考业界标准；
    * > 1.0：愿景型目标；

5. 评分规则

    能者多得、多劳多得，以单项KR为计分单位，累计总分。
    
    单项KR评分公式：(工作时长 / 8) * 0.3 * 权重值 * 完成度。
    
    其中，工时时长以 **小时**为单位，计算时转换为工作日（每个工作日为8小时）；每个工作日的基础分值为0.3；完成度取值范围：[0.00, 1.00]，如果完成情况超出预期，取值也可以大于1.00。

## 基于Gitlab的OKR实施方法

* 使用Milestone表示 **目标（O）**
* 使用Issue表示 **（关键结果）**
* 使用Label表示 **关键结果权重值**
* 使用/spend表示 **工时**
* 使用百分比表示 **完成度**

1. 创建Project（项目）

    ![project](https://yurun-blog.oss-cn-beijing.aliyuncs.com/OKR%E5%AE%9E%E6%96%BD%E6%96%B9%E6%B3%95%E8%AE%BA/project.png)
    
2. 创建Milestone（里程碑）

   Milestone（里程碑）用于目标（O）；其中， 标题（Title）用于说明目标名称，通常为项目名称；描述（Description）用于详细说明项目需要完成的系统功能、服务模块等；示例如下：

    ![milestone](https://yurun-blog.oss-cn-beijing.aliyuncs.com/OKR%E5%AE%9E%E6%96%BD%E6%96%B9%E6%B3%95%E8%AE%BA/milestone.png)
    
3. 创建Label（标签）

    Label（标签）用于权重值，如下：
    
    ![label](https://yurun-blog.oss-cn-beijing.aliyuncs.com/OKR%E5%AE%9E%E6%96%BD%E6%96%B9%E6%B3%95%E8%AE%BA/label.png)
    
4. 创建Issue（关键结果）
    
    Issue（事务）用于关键结果（KR）；其中，标题（Title）用于说明关键结果名称，通常为需要开发的服务模块或系统功能名称；描述（Description）用于详细说明关键结果的可衡量指标，用于团队负责人或相关业务同学可清晰判断关键结果的完成情况；执行人（Assignee）用于说明关键结果具体的执行人；里程碑（Milestone）用于说明关键结果对应的目标；Labels（标签）用于说明关键结果对应的权重值；示例如下；
    
    ![issue](https://yurun-blog.oss-cn-beijing.aliyuncs.com/OKR%E5%AE%9E%E6%96%BD%E6%96%B9%E6%B3%95%E8%AE%BA/issue.png)
    
5. 更新Issue（关键结果）进度

    Issue进行过程中，使用评论（commet）记录进展详情及工作时长，如下：
    
    ![comment](https://yurun-blog.oss-cn-beijing.aliyuncs.com/OKR%E5%AE%9E%E6%96%BD%E6%96%B9%E6%B3%95%E8%AE%BA/comment.png)
    
    **注：** 工作时长使用Gitlab quick actions中的 **/spend** 进行记录。
    
    Issue进行完成之后，需要团队负责人或相关业务同学使用评论确认完成度，如下：
    
    ![comment](https://yurun-blog.oss-cn-beijing.aliyuncs.com/OKR%E5%AE%9E%E6%96%BD%E6%96%B9%E6%B3%95%E8%AE%BA/comment_percent.png)
    
    Issue完成度确认之后，即可关闭Issue。
    
    **注：** 已关闭Issue已关闭如需要更新，需要重新开启，且更新完成之后需要重新确认完成度之后才可关闭。
